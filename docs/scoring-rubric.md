# Discovery & Enrichment Scoring Rubric

How a candidate becomes a qualified lead.

---

## The composite score

A candidate gets a 0–100 score. The default cutoff is **60**. Below
that, `funnel_state = REJECTED:enrichment` and the candidate is out.

```
score = (
    audience_niche_match    * 0.30 +
    language_geography_fit  * 0.20 +
    real_engagement_rate    * 0.20 +
    follower_in_band        * 0.15 +
    posting_cadence         * 0.10 +
    cross_platform_presence * 0.05
) * 100
```

Each sub-score is normalized 0–1 before weighting. Below: how each one
is computed and why it's weighted the way it is.

---

## Audience-niche match (30%)

The single biggest predictor of conversion. Highest weight by far.

**How.** Claude classifies the creator's last 30 posts (or video
descriptions) against the client's target niche taxonomy. Output is a
0–1 confidence score that the creator's audience is genuinely in-niche.

**Niche taxonomy example for crypto exchange:**
- Trading / technical analysis (TA chartists, market commentary)
- Spot / long-term investing (HODL community)
- DeFi / on-chain (yield farming, liquidity provision)
- Derivatives / leverage trading
- Stablecoin / RWA / fiat on-ramp
- Education / "explain like I'm five"

A creator who posts 80% TA and 20% lifestyle for a TA-targeted campaign
scores ~0.85. A creator who posts 50% crypto-adjacent lifestyle, 30% NFT
hype, 20% trading scores ~0.4 — they're crypto-aware but not the right
audience for a derivatives product.

**Why 30%.** Niche match is the primary lever for conversion. A 50K
in-niche creator beats a 500K out-of-niche creator on every metric
that matters: click-through, registration, deposit, retention.

---

## Language & geography fit (20%)

Closely behind niche match. Wrong language = zero conversion.

**How.**
- Bio language detection (heuristic + Claude fallback)
- Last 30 posts' language ratio
- Comments-section language ratio (often more telling than the creator's
  own posts — a Mandarin creator with a 70% English comment section is
  signaling a different audience than they appear to)
- Geography signals: bio mentions, geo-tagged posts, sponsor history

For a Taiwan-market campaign we require ≥70% Traditional Chinese in
either the creator's posts or their comment threads. Simplified Chinese
is flagged for review (mainland audience risk).

---

## Real engagement rate (20%)

The "is this audience actually paying attention" check.

**How.**
```
engagement_rate = (avg_likes + avg_replies) / follower_count
```

Computed across the last 30 posts. The thresholds vary by platform —
Instagram is generally 2–4×% more engaging than X for the same niche —
so we normalize against per-platform medians, not absolute numbers.

**Why this matters more than follower count.** Bought followers don't
engage. A creator with 200K followers and a 0.3% engagement rate has
~600 active fans. A creator with 30K followers and a 5% engagement rate
has 1,500 active fans. The smaller account converts better.

**Anti-fraud check.** If the engagement rate is suspiciously high
(>15% on X), we run a coordinated-engagement detection pass. Sudden
engagement spikes that don't match the post's organic reach signal
engagement-pod / bot activity.

---

## Follower in-band (15%)

The Goldilocks zone. Too small = not worth the operational cost. Too
big = priced out, distracted, or already taken.

**Default bands.**
- Spot trading: 10K – 200K
- Derivatives: 25K – 300K
- DeFi / on-chain: 5K – 100K (smaller niche, higher per-follower value)
- Stablecoin / RWA: 15K – 250K

These are starting bands; they get tuned per client. A protocol with a
small marketing budget benefits from 5K–50K creators (cheaper, more
attention per dollar). An exchange with a brand-awareness goal can
absorb 100K–1M creators.

---

## Posting cadence (10%)

A creator who hasn't posted in 30 days is functionally inactive. We
filter them out hard.

**Scoring.**
- Posted in last 7 days: 1.0
- Posted in last 14 days: 0.6
- Posted in last 30 days: 0.2
- Otherwise: 0.0

Combined with consistency: a creator who posts daily for 6 days then
goes quiet for a month scores worse than one who posts 3× a week
reliably for the last 90 days.

---

## Cross-platform presence (5%)

Lower weight, useful tiebreaker.

**How.** A creator with linked, active presence on 2+ platforms is
worth slightly more — partnership content can be repurposed across
their channels with minimal extra cost.

**Detection.** Bio links, "find me on" mentions, identity-confirmable
matches across platforms.

---

## Tuning the rubric per client

The weights above are defaults, not commandments. Per-client tuning is
expected:

- A B2B SaaS company doing thought-leadership outreach might weight
  *audience-niche match* at 50% and barely care about engagement rate.
- A consumer e-commerce brand doing fast performance campaigns might
  weight *real engagement rate* at 30% and de-emphasize cadence.
- A regulated fintech might add a synthetic seventh factor — *prior
  disclosure quality* — and weight it at 10%.

The `weights.yaml` file is the single source of truth. Change the
file, redeploy the orchestrator, the rubric updates with no code
change.

---

## Calibration

Every two weeks the rubric is calibrated against actual conversion data
from the funnel tracker:

1. Pull the last 60 days of activated partners.
2. Compare their composite scores at outreach time to their actual
   conversion outcomes.
3. Identify weight imbalances (e.g., are 70-scoring creators converting
   *better* than 90-scoring ones? — a sign the niche-match weight is
   too dominant).
4. Adjust weights, document the change, redeploy.

This is the part of the system that gets better over time. The
architecture stays static; the rubric learns.
