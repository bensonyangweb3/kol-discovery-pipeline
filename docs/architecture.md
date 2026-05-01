# Architecture · KOL Discovery Pipeline

A deeper look at the design decisions inside the pipeline. Read the
[main README](../README.md) first for the high-level picture.

---

## Module boundaries

```
                             ┌─────────────────────────┐
                             │      Orchestrator       │
                             │  (cron-driven, async)   │
                             └─────┬──────────────┬────┘
                                   │              │
                  ┌────────────────┘              └────────────────┐
                  ▼                                                ▼
         ┌─────────────────┐                              ┌─────────────────┐
         │  PlatformScraper│   ←─ each platform impls ─→  │   StateStore    │
         │   (Playwright)  │       a uniform interface     │ (SQLite/PG)     │
         └────────┬────────┘                              └────────┬────────┘
                  │                                                │
                  ▼                                                │
         ┌─────────────────┐                                       │
         │   Enricher      │   ←─ Claude API, cached ──┐           │
         │  (LLM scorer)   │                           │           │
         └────────┬────────┘                           ▼           │
                  │                          ┌──────────────────┐  │
                  ▼                          │  Compliance Gate │  │
                                             └────────┬─────────┘  │
                                                      │            │
                                                      ▼            │
                                             ┌──────────────────┐  │
                                             │   Outreach       │  │
                                             │  (channel-aware) │  │
                                             └────────┬─────────┘  │
                                                      │            │
                                                      ▼            ▼
                                                ┌────────────────────┐
                                                │  Funnel Tracker    │
                                                │  + Slack alerts    │
                                                └────────────────────┘
```

Each box is a Python module with a clean interface. The orchestrator
schedules jobs and routes data between them but never reaches into a
module's internals.

---

## The `PlatformScraper` interface

Adding support for a new platform = writing one module.

```python
from typing import Protocol, AsyncIterator

class CandidateRecord:
    platform: str
    handle: str
    follower_count: int
    bio: str
    recent_posts: list[str]
    engagement_signals: dict
    raw_metadata: dict

class PlatformScraper(Protocol):
    """Every platform module implements this contract."""

    name: str  # e.g. "x", "youtube", "instagram", "telegram"

    async def discover(
        self,
        niche_keywords: list[str],
        language: str,
        min_followers: int,
        max_followers: int,
    ) -> AsyncIterator[CandidateRecord]: ...

    async def fetch_engagement(
        self, handle: str, posts: int = 30
    ) -> dict: ...
```

Concrete implementations live in `pipeline/discovery/{x,youtube,instagram,telegram}.py`.
Each one handles its own platform-specific quirks (auth, anti-bot,
rate-limit cadence) but exposes the same surface.

---

## Enrichment scoring

Stored as YAML-configurable weights so non-engineers can tune the model.

```yaml
# pipeline/enrichment/weights.yaml
audience_niche_match:    0.30   # Claude classifies last 30 posts
language_geography_fit:  0.20   # Bio + post language detection
real_engagement_rate:    0.20   # (likes + replies) / followers
follower_in_band:        0.15   # configurable per niche
posting_cadence:         0.10   # active in last 14 days
cross_platform_presence: 0.05   # linked accounts
```

A composite score 0–100 is written back to the candidate record. Anything
below the configured threshold (default 60) is marked `rejected:enrichment`
and skipped. Above the threshold flows to the compliance gate.

---

## The compliance gate

Six checkpoints, each one binary. Any failure kills the candidate before
outreach.

| Checkpoint            | Mechanism            | Configurable per client       |
| --------------------- | -------------------- | ----------------------------- |
| Jurisdiction          | Audience-country detection + allowlist | yes |
| Sanctions screening   | OFAC / EU SDN list lookup            | no  |
| Regulated content     | Claude review of last 30 posts       | yes (rules per regulator)     |
| Brand safety          | Claude review + scandal lookup       | yes |
| Disclosure standards  | Past disclosure pattern check        | yes |
| Conflict of interest  | Active competitor partnerships       | yes |

The gate logs every rejection with the reason and the evidence (post URL
+ excerpt). This becomes the audit trail Legal asks for during reviews.

---

## State machine — funnel tracking

Every candidate is in exactly one funnel state at any time.

```
        DISCOVERED
            │
            ▼
        ENRICHED ───► REJECTED:enrichment
            │
            ▼
       COMPLIANT ───► REJECTED:compliance
            │
            ▼
        OUTREACHED ──► NO_REPLY (after T days)
            │
            ▼
        REPLIED
            │
            ▼
        ONBOARDING ──► DROPPED
            │
            ▼
            ACTIVATED ─────► CHURNED
                │
                ▼
             RETAINED
```

Transitions are recorded with timestamps so per-partner funnel velocity
becomes a first-class metric. This is the data view that lets you defund
the bottom-quartile partners and double down on the top decile.

---

## Outreach channel matrix

| Channel              | Best for                          | Throttle policy                    |
| -------------------- | --------------------------------- | ---------------------------------- |
| Email (Gmail API)    | Established creators with linked websites | 50/day, 30s–5min jitter     |
| Instagram DM         | Lifestyle-adjacent creators       | 20/day, 1–4 min jitter; warm DMs only |
| FB Messenger         | Older creator demographics        | 15/day, similar pacing             |
| YouTube comments     | Breaking through to email-saturated creators | 10/day, hand-curated tone  |
| Telegram             | Asian-market traders              | 30/day, public groups + invite-only |

Each channel sender is its own module under `pipeline/outreach/`. They
share a `send(message, recipient, channel_metadata)` signature.

---

## Storage schema (SQLite default)

The schema is intentionally narrow — five tables, normalized, with the
funnel state denormalized onto the candidate row for query speed.

```sql
CREATE TABLE candidate (
    id              TEXT PRIMARY KEY,        -- platform:handle
    platform        TEXT NOT NULL,
    handle          TEXT NOT NULL,
    follower_count  INTEGER,
    bio             TEXT,
    discovered_at   DATETIME,
    enrichment_score REAL,
    funnel_state    TEXT,                    -- DISCOVERED, ENRICHED, ...
    funnel_state_at DATETIME,
    UNIQUE(platform, handle)
);

CREATE TABLE outreach_send (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    candidate_id    TEXT REFERENCES candidate(id),
    channel         TEXT,                    -- email, ig_dm, ...
    sent_at         DATETIME,
    message_body    TEXT,
    reply_received  BOOLEAN DEFAULT 0,
    reply_at        DATETIME
);

CREATE TABLE compliance_check (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    candidate_id    TEXT REFERENCES candidate(id),
    checkpoint      TEXT,                    -- jurisdiction, sanctions, ...
    passed          BOOLEAN,
    reason          TEXT,
    evidence_url    TEXT,
    checked_at      DATETIME
);

CREATE TABLE funnel_event (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    candidate_id    TEXT REFERENCES candidate(id),
    event_type      TEXT,                    -- click, register, deposit, ...
    event_at        DATETIME,
    metadata_json   TEXT
);

CREATE TABLE enrichment_cache (
    cache_key       TEXT PRIMARY KEY,        -- hash(handle + content version)
    response_json   TEXT,
    cached_at       DATETIME
);
```

For Postgres deployments the same schema applies; we add appropriate
indexes (`candidate.funnel_state`, `outreach_send.candidate_id`,
`funnel_event.candidate_id + event_at`) and convert SQLite-specific
types.

---

## Operational concerns

**Anti-bot resilience.** Playwright with stealth-plugin baseline; per-
platform retry budgets; if a session gets challenged, the worker
gracefully degrades to a longer scan window the next cycle.

**Cost control.** Enrichment is the dominant cost. The cache key is
`(handle, content_version)` where content_version increments only when
new posts are detected. Re-enriching the same candidate with the same
content is a no-op.

**Concurrency.** Async at the worker level (Playwright supports it
natively); each platform scraper runs in its own task. Compliance gate
and enrichment are LLM-bound, so we batch and respect Claude's rate
limits.

**Observability.** Every stage emits a structured log line with the
candidate ID, stage, duration, and outcome. Easy to pipe into any
log aggregator. Slack alerts only on actionable events (replies,
compliance rejections that look like false positives, hard errors).

**Failure modes.** Each stage is idempotent — re-running on a candidate
that's already passed is a no-op. The orchestrator can crash mid-cycle
and resume cleanly on the next run.

---

## What's intentionally not in this repo

- The actual scoring prompts (proprietary; per-client tuned)
- The compliance rule library (jurisdiction-specific; per-client tuned)
- The outreach message templates (per-client; would identify clients)
- Any client data, ever
- Production deployment scripts that include client-specific secrets

If you want any of the above for your own deployment, that's a paid
engagement. See [Hire me](../README.md#hire-me).
