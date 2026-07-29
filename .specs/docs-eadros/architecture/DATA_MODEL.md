# EADROS Data Model

**Store:** one SQLite file, WAL mode, `STRICT` tables
([ADR-0016](../adr/ADR-0016-local-first-single-file-store.md)). Identifiers are **ULIDs** as `TEXT`
(sortable by creation time, no extension required). Timestamps are **ISO-8601 UTC strings** —
readable in a diff, sortable lexically, and honest about the timezone. Structured columns are `TEXT`
guarded by `json_valid()`.

Five design rules run through the schema, each fixing a defect that would otherwise surface in
production rather than in review:

1. **A campaign and a post have different lifecycles.** One story published to four channels is one
   campaign and four posts, each with its own gate verdict, approval and outcome. Modelling this as
   two loosely-related status columns is what makes a partial publish unrepresentable.
2. **Anything that happened is append-only.** Approvals, gate verdicts, transitions, metrics and
   events are never updated in place. State is derived from the ledger, not overwritten on top of it.
3. **Every published post carries a unique idempotency key.** The database, not the application,
   refuses the second insert.
4. **Rules are denormalised at the moment they were applied.** Channel profiles and voice profiles
   change; an audit that reads today's profile to explain last month's post is not an audit.
5. **An absent measurement is not a zero.** Gaps are recorded as gaps
   ([ADR-0015](../adr/ADR-0015-attribution-methodology.md)).

---

## Pipeline

```sql
CREATE TABLE repos (
  id            TEXT PRIMARY KEY,
  slug          TEXT NOT NULL UNIQUE,        -- owner/name
  visibility    TEXT NOT NULL CHECK (visibility IN ('public','private')),
  created_at    TEXT NOT NULL
) STRICT;
-- Present from day one although v1 is single-repo: the V2 workspace then costs a data move
-- rather than a redesign (ADR-0016).

CREATE TABLE candidates (
  id            TEXT PRIMARY KEY,
  repo_id       TEXT NOT NULL REFERENCES repos(id),
  mined_at      TEXT NOT NULL,
  source_refs   TEXT NOT NULL CHECK (json_valid(source_refs)),  -- [{kind:'commit',ref:'a1b2c3d'}]
  score         REAL NOT NULL,
  signals       TEXT NOT NULL CHECK (json_valid(signals)),      -- {gate_intercept:0.4,...} incl. zeroed
  archetypes    TEXT NOT NULL CHECK (json_valid(archetypes)),   -- eligible AFTER consent filtering
  content_hash  TEXT NOT NULL,
  dedup_verdict TEXT NOT NULL CHECK (dedup_verdict IN ('new','duplicate','supersedes')),
  supersedes_post_id TEXT REFERENCES posts(id),
  excluded_reason TEXT,                       -- non-null for candidates mine scored but cut
  UNIQUE (repo_id, content_hash)
) STRICT;
-- Cut candidates are STORED, not discarded: the rejection set is where a maintainer notices the
-- weights are wrong, and it is the corpus /eadros eval scores recall against.

CREATE TABLE campaigns (
  id            TEXT PRIMARY KEY,
  repo_id       TEXT NOT NULL REFERENCES repos(id),
  candidate_id  TEXT NOT NULL REFERENCES candidates(id),
  archetype     TEXT NOT NULL,
  narrative_hook TEXT NOT NULL,
  state         TEXT NOT NULL,                -- derived aggregate; see STATE_MACHINE.md
  opened_at     TEXT NOT NULL,
  closed_at     TEXT
) STRICT;

CREATE TABLE drafts (
  id            TEXT PRIMARY KEY,
  campaign_id   TEXT NOT NULL REFERENCES campaigns(id),
  channel       TEXT NOT NULL,
  locale        TEXT NOT NULL,
  iteration     INTEGER NOT NULL DEFAULT 1,   -- capped by budget.max_reviewer_iterations
  body          TEXT NOT NULL,
  content_hash  TEXT NOT NULL,
  voice_profile TEXT NOT NULL,                -- DENORMALISED: the profile as applied, not as it is now
  model_tier    TEXT NOT NULL CHECK (model_tier IN ('cheap','mid','strong')),
  model_id      TEXT NOT NULL,                -- resolved by the provider profile at generation time
  prompt_version TEXT NOT NULL,
  input_tokens  INTEGER NOT NULL,
  output_tokens INTEGER NOT NULL,
  cost_micros   INTEGER NOT NULL,             -- integer minor units; never float money
  created_at    TEXT NOT NULL,
  UNIQUE (campaign_id, channel, locale, iteration)
) STRICT;
-- model_id + prompt_version are what make a quality regression debuggable: without them you cannot
-- answer "what changed" after a bad week (ADR-0013).
```

## Governance

```sql
CREATE TABLE claims (
  id            TEXT PRIMARY KEY,
  draft_id      TEXT NOT NULL REFERENCES drafts(id),
  span_start    INTEGER NOT NULL,
  span_end      INTEGER NOT NULL,
  claim_text    TEXT NOT NULL,
  source_kind   TEXT NOT NULL CHECK (source_kind IN ('commit','pr','file_line','benchmark','issue')),
  source_ref    TEXT NOT NULL,
  resolved      INTEGER NOT NULL CHECK (resolved IN (0,1)),
  resolved_at   TEXT,
  resolution_note TEXT
) STRICT;
-- The claim discipline of ADR-0014 made concrete. An unresolved claim FAILS its draft; it is not
-- softened into vagueness, which is what a model asked to "be careful" produces instead.

CREATE TABLE gate_verdicts (
  id            TEXT PRIMARY KEY,
  subject_kind  TEXT NOT NULL CHECK (subject_kind IN ('candidate','draft')),
  subject_id    TEXT NOT NULL,
  pass          TEXT NOT NULL CHECK (pass IN ('input','output')),
  stage         TEXT NOT NULL CHECK (stage IN
                  ('secrets','deny_terms','paths','diff_cap','embargo','taint','claims','voice_lint')),
  verdict       TEXT NOT NULL CHECK (verdict IN ('pass','block')),
  reason        TEXT,
  offending_span TEXT,
  evaluated_at  TEXT NOT NULL,
  false_positive INTEGER CHECK (false_positive IN (0,1))   -- set by a human on review
) STRICT;
-- Two passes per ADR-0014: 'input' before any model call (this is what stops injection), 'output'
-- before dispatch. `false_positive` exists so /eadros audit can measure the rate — an unmeasured
-- gate is a gate on its way to being switched off.

CREATE TABLE approvals (
  id            TEXT PRIMARY KEY,
  draft_id      TEXT NOT NULL REFERENCES drafts(id),
  approver      TEXT NOT NULL,
  approved_at   TEXT NOT NULL,
  mode          TEXT NOT NULL CHECK (mode IN ('edit-required','as-is')),
  hash_before   TEXT NOT NULL,                -- the generated draft
  hash_after    TEXT NOT NULL,                -- the text actually approved
  final_body    TEXT NOT NULL,
  surface       TEXT NOT NULL CHECK (surface IN ('cli','dashboard','discord','telegram')),
  CHECK (mode = 'as-is' OR hash_before <> hash_after)
) STRICT;
-- The CHECK constraint IS the edit-required policy: an unedited approval cannot be recorded unless
-- it is explicitly marked as-is, so the rubber stamp leaves a mark. Governance enforced by the
-- database outlives governance enforced by intention.

CREATE TABLE rejections (
  id            TEXT PRIMARY KEY,
  draft_id      TEXT NOT NULL REFERENCES drafts(id),
  rejected_by   TEXT NOT NULL,
  rejected_at   TEXT NOT NULL,
  reason        TEXT NOT NULL                 -- feeds the /eadros eval corpus
) STRICT;
```

## Publishing

```sql
CREATE TABLE posts (
  id              TEXT PRIMARY KEY,
  campaign_id     TEXT NOT NULL REFERENCES campaigns(id),
  draft_id        TEXT NOT NULL REFERENCES drafts(id),
  approval_id     TEXT NOT NULL REFERENCES approvals(id),   -- NOT NULL: no post without an approval
  channel         TEXT NOT NULL,
  locale          TEXT NOT NULL,
  tier_at_publish TEXT NOT NULL CHECK (tier_at_publish IN ('auto','assisted','draft')),
  policy_verified_on TEXT NOT NULL,           -- the profile's verification date, as applied
  idempotency_key TEXT NOT NULL,
  state           TEXT NOT NULL,              -- see STATE_MACHINE.md
  scheduled_for   TEXT,
  published_at    TEXT,
  external_url    TEXT,
  tracking_url    TEXT,                       -- the owned redirect (ADR-0015)
  UNIQUE (idempotency_key)
) STRICT;
-- `approval_id NOT NULL` and `UNIQUE(idempotency_key)` are the two invariants that matter most, and
-- both are enforced by the engine rather than by the publisher's control flow. tier_at_publish and
-- policy_verified_on are denormalised deliberately: profiles change, and an audit that explains
-- last month's post using this month's rules is not an audit.

CREATE TABLE outbox (
  id              TEXT PRIMARY KEY,
  post_id         TEXT NOT NULL REFERENCES posts(id),
  intent_at       TEXT NOT NULL,              -- committed BEFORE the platform call
  outcome         TEXT CHECK (outcome IN ('success','failure','unknown')),
  outcome_at      TEXT,
  attempts        INTEGER NOT NULL DEFAULT 0,
  next_attempt_at TEXT,
  last_error      TEXT,
  dead_lettered   INTEGER NOT NULL DEFAULT 0 CHECK (dead_lettered IN (0,1))
) STRICT;
-- `outcome = 'unknown'` is the row that matters: the process died between the call and the record.
-- `publish --reconcile` asks the platform whether the post exists. It never retries blind — that
-- is precisely how a timeout becomes a double post.

CREATE TABLE retractions (
  id              TEXT PRIMARY KEY,
  post_id         TEXT NOT NULL REFERENCES posts(id),
  class           TEXT NOT NULL CHECK (class IN ('leak','false','voice','duplicate')),
  action_taken    TEXT NOT NULL CHECK (action_taken IN ('deleted','unpublished','corrected','comment_only')),
  removal_possible INTEGER NOT NULL CHECK (removal_possible IN (0,1)),
  gate_verdict_id TEXT REFERENCES gate_verdicts(id),   -- the verdict that PASSED this content
  retracted_by    TEXT NOT NULL,
  retracted_at    TEXT NOT NULL,
  note            TEXT
) STRICT;
-- gate_verdict_id closes the learning loop: a leak that reached publication is a gate defect, and
-- this column names the verdict that has to become a test case.

CREATE TABLE post_transitions (
  id            TEXT PRIMARY KEY,
  post_id       TEXT NOT NULL REFERENCES posts(id),
  from_state    TEXT,                         -- NULL only for the first entry
  to_state      TEXT NOT NULL,
  trigger       TEXT NOT NULL,
  actor         TEXT NOT NULL,                -- a person's identity, or 'agent' / 'system'
  guard_results TEXT NOT NULL CHECK (json_valid(guard_results)),
  correlation_id TEXT NOT NULL,
  at            TEXT NOT NULL
) STRICT;
-- The ledger, written in the SAME TRANSACTION as the state change (ADR-0016). posts.state is a
-- cache of the last row here, not an independent fact: a state column that disagrees with its own
-- history is a bug the schema should surface, never a discrepancy reconciled by hand.
-- guard_results records WHICH guards were evaluated and what they returned — evidence, not honour,
-- following the EADOS delivery_state.checkpoints pattern.
```

## Measurement

```sql
CREATE TABLE post_metrics (
  post_id       TEXT NOT NULL REFERENCES posts(id),
  metric_date   TEXT NOT NULL,                -- YYYY-MM-DD
  captured_at   TEXT NOT NULL,
  views         INTEGER,
  reactions     INTEGER,
  comments      INTEGER,
  clicks        INTEGER,                      -- from the owned redirect, not the platform
  is_gap        INTEGER NOT NULL DEFAULT 0 CHECK (is_gap IN (0,1)),
  PRIMARY KEY (post_id, metric_date)
) STRICT;

CREATE TABLE repo_traffic (
  repo_id       TEXT NOT NULL REFERENCES repos(id),
  metric_date   TEXT NOT NULL,
  captured_at   TEXT NOT NULL,
  views         INTEGER, unique_visitors INTEGER,
  clones        INTEGER, unique_cloners   INTEGER,
  referrers     TEXT CHECK (referrers IS NULL OR json_valid(referrers)),
  is_gap        INTEGER NOT NULL DEFAULT 0 CHECK (is_gap IN (0,1)),
  PRIMARY KEY (repo_id, metric_date)
) STRICT;
-- Two tables, deliberately not joined into one: repo traffic is REPO-level and cannot be
-- attributed per post (ADR-0015). Merging them would invite exactly the causal claim that ADR-0015
-- forbids. `is_gap` marks a day the snapshot missed — NULL metrics on a gap row read as "not
-- measured", never as zero, because the Traffic API's ~14-day window makes the loss permanent.

CREATE TABLE budget_ledger (
  id            TEXT PRIMARY KEY,
  repo_id       TEXT NOT NULL REFERENCES repos(id),
  period_start  TEXT NOT NULL,
  stage         TEXT NOT NULL CHECK (stage IN ('triage','drafting','review','other')),
  cost_micros   INTEGER NOT NULL,
  recorded_at   TEXT NOT NULL
) STRICT;
-- Per-stage, so /eadros digest can answer "what did this week cost and which stage spent it" —
-- the question that decides whether the tool stays installed (ADR-0013).
```

## Knowledge base & event log

```sql
CREATE TABLE kb_documents (
  id            TEXT PRIMARY KEY,
  repo_id       TEXT NOT NULL REFERENCES repos(id),
  kind          TEXT NOT NULL CHECK (kind IN ('readme','adr','doc','past_post','spec')),
  ref           TEXT NOT NULL,
  content_hash  TEXT NOT NULL,
  indexed_at    TEXT NOT NULL,
  UNIQUE (repo_id, ref)
) STRICT;

CREATE VIRTUAL TABLE kb_fts USING fts5(body, document_id UNINDEXED);

CREATE TABLE kb_embeddings (
  chunk_id      TEXT PRIMARY KEY,
  document_id   TEXT NOT NULL REFERENCES kb_documents(id),
  embedding     BLOB NOT NULL,
  model_id      TEXT NOT NULL
) STRICT;
-- FTS5 plus an exhaustive cosine scan. At a few thousand chunks this completes in microseconds;
-- a dedicated vector database here would be answering a question nobody asked (ADR-0016).

CREATE TABLE events (
  id              TEXT PRIMARY KEY,           -- CloudEvents `id`; the deduplication key
  type            TEXT NOT NULL,              -- dev.eadros.<domain>.<event>.v<N>
  source          TEXT NOT NULL,
  subject         TEXT,
  time            TEXT NOT NULL,
  correlation_id  TEXT NOT NULL,
  causation_id    TEXT,
  taint           TEXT NOT NULL DEFAULT 'trusted' CHECK (taint IN ('trusted','untrusted')),
  data            TEXT NOT NULL CHECK (json_valid(data)),
  handled_at      TEXT
) STRICT;
-- Persisted in the same file, and written in the SAME TRANSACTION as the state change it
-- accompanies (ADR-0016). An audit trail committed separately from the state it describes can
-- disagree with it, and does. See EVENTS.md.
```

## Indices

```sql
CREATE INDEX idx_candidates_score   ON candidates(repo_id, score DESC, mined_at DESC);
CREATE INDEX idx_posts_state        ON posts(state, scheduled_for);
CREATE INDEX idx_outbox_pending     ON outbox(next_attempt_at) WHERE outcome IS NULL;
CREATE INDEX idx_gate_stage         ON gate_verdicts(stage, verdict, evaluated_at);
CREATE INDEX idx_events_correlation ON events(correlation_id, time);
CREATE INDEX idx_metrics_gaps       ON post_metrics(metric_date) WHERE is_gap = 1;
```

## Invariants the schema enforces

| Invariant | Mechanism |
|---|---|
| No post exists without an approval | `posts.approval_id NOT NULL` |
| No post is published twice | `UNIQUE (posts.idempotency_key)` |
| An unedited approval cannot hide | `CHECK (mode = 'as-is' OR hash_before <> hash_after)` |
| The same story is not mined twice | `UNIQUE (candidates.repo_id, content_hash)` |
| A metric gap is not a zero | `is_gap` + nullable metric columns |
| Post attribution is never mixed with repo traffic | Two tables, no join key |

The remaining invariants — *no dispatch while paused*, *a `draft`-tier post never dispatches*, *a
transition chain is contiguous* — are behavioural rather than structural and are enforced by the
state machine and its property tests. See [`STATE_MACHINE.md`](STATE_MACHINE.md).
