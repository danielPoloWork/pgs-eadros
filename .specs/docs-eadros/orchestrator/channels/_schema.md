# Channel profile schema

One file per destination: `channels/<name>.yaml`. A channel profile is the machine-readable
record of **what a platform actually permits**, separate from what a maintainer wants. It is the
sibling of an EADOS language profile, and it carries the same stance:

> **There is no unsupported channel, only a channel not yet profiled.**

If a maintainer names a destination with no profile, that is the normal path, not an error:
copy `_template.yaml`, verify the platform's automation terms, record `verified_on`, add an ADR,
then resume the interview. **Never hardcode a platform's rules into an adapter to skip this** —
rules that live in code cannot be re-verified, and an unverifiable rule is how an account gets
banned.

## The tier is the point

`tier` is the field the rest of the system reads. It is **derived here and stated to the
maintainer, never offered as a choice** ([ADR-0011](../../adr/ADR-0011-channel-capability-tiers.md)):

| Tier | Meaning | Publisher behaviour |
|---|---|---|
| `auto` | A documented write API, and automated posting is permitted by the platform's terms | Dispatches on approval |
| `assisted` | A write API exists but is constrained — quota, app review, expiring credentials | Dispatches under a metered budget; refuses when quota is spent |
| `draft` | **No lawful automation path.** No write API, or automated posting violates the platform's rules | Never dispatches. Prepares the payload and a pre-filled composer link; a human posts and pastes the URL back |

A `draft` tier is not a missing feature. It is the correct, permanent answer for platforms whose
value depends on submissions being human — and the hand-off still closes the analytics loop,
because the returned URL is what the metrics collector tracks.

## Fields

```yaml
channel:        devto                 # profile key, matches the filename
display_name:   "DEV Community"
tier:           auto                  # auto | assisted | draft — DERIVED, see above
tier_reason:    ""                    # one line, stated to the maintainer at Phase 2 confirmation

policy:                               # what the platform permits — the half that must be re-verified
  automation_permitted: true
  self_promotion:       none          # none | subreddit-level | discouraged | prohibited
  ban_scope:            account       # account | domain — domain bans outlive the account and
                                      # are the reason an upward tier override is an ADR, not a flag
  terms_url:            ""
  verified_on:          "2026-07-28"  # /eadros doctor warns past 90 days; the interview
                                      # surfaces a stale date BEFORE Phase 2 settles
  notes:                ""

api:
  write:          true                # true | false | conditional
  auth:           api_key             # api_key | oauth2 | webhook | none
  requires_app_review: false
  token_lifetime_days: null           # null = non-expiring; drives the doctor's expiry warning
  endpoint_hint:  ""                  # documentation pointer, not a hardcoded URL
  delete:         supported           # supported | edit-only | unsupported — the retract runbook
                                      # ([/eadros retract]) branches on this, so it is not optional

limits:
  max_chars:      null                # null = no practical limit
  markdown:       full                # full | limited | none
  code_blocks:    fenced              # fenced | indented | image-only | unsupported
  images:         url                 # url | upload | unsupported
  links:          { count: null, penalised: false }   # some feeds suppress link posts
  tags:           { max: 4, style: lowercase-nospace }
  rate:           { unit: month, quota: null }

cadence:                              # profile defaults; the interview may lower, never raise
  max_per_week:   1
  min_gap_hours:  72

format:
  front_matter:   true
  canonical_url:  supported           # supported | unsupported — where supported it is set to the
                                      # project's own docs, so syndication does not outrank the source
  series:         supported
  thread:         unsupported         # supported = the copywriter may emit a multi-part draft

attribution:
  utm:                supported
  referrer_preserved: true            # false -> per-post attribution needs an owned redirect;
                                      # the analytics engine must not claim a causal link (ADR-0015)

handoff:                              # REQUIRED when tier == draft
  method:         prefilled_composer  # prefilled_composer | clipboard | manual
  url_template:   ""
  returns_url:    true                # the human pastes the live URL back to close the loop
```

## Shipped profiles

| Profile | Tier | Why |
|---|---|---|
| [`devto.yaml`](devto.yaml) | `auto` | Documented write API, automation permitted |
| [`linkedin.yaml`](linkedin.yaml) | `assisted` | Write access is gated on app review and short-lived credentials |
| [`hackernews.yaml`](hackernews.yaml) | `draft` | No write API, and submissions are expected to be human |

These three are seeds chosen to demonstrate all three tiers, not the allowed set. Hashnode,
Mastodon, Discord, GitHub Releases, X and Reddit are the next profiles to author.
