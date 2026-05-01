# Compliance Gate · Six Checkpoints

The single most expensive lesson Web3 BD teams learn the hard way:
*your first compliance review of a creator must happen before outreach,
not after their first paid post is live.*

This document walks through each checkpoint, what it catches, and how
it's implemented.

---

## 1. Jurisdiction

**Catches:** Creators whose primary audience sits in a country your
operating license does not cover. (For a Taiwan-licensed exchange:
audiences whose dominant location is mainland China, North Korea, Iran,
or any sanction-listed jurisdiction.)

**Mechanism.** Audience-country distribution is inferred from comment
language patterns, comment-author profile metadata, and geo-tag
exposure on recent posts. We require ≥60% audience overlap with an
allowlisted country to pass.

**Configurable.** Per-client allowlist of acceptable audience countries.

---

## 2. Sanctions screening

**Catches:** Creators on OFAC SDN, EU sanctions lists, or comparable
national lists. Also catches creators whose listed business addresses
or known affiliations land inside sanctioned entities.

**Mechanism.** Programmatic lookup against published sanctions lists at
discovery time. The lookup also fires before outreach (lists update
weekly; a creator who passed a month ago may not pass today).

**Not configurable.** This gate cannot be loosened by a client.

---

## 3. Regulated content history

**Catches:** Creators whose past content suggests they will run afoul of
the regulator that governs your product. For a crypto exchange in
Taiwan that means: unregistered securities promotion, leverage
recommendations without risk disclosures, fraud-adjacent endorsements,
or "guaranteed return" language.

**Mechanism.** Claude reviews the most recent 30 posts (or video
descriptions / transcripts) against a per-regulator rule library. A
flagged post returns the URL and a quoted excerpt as evidence.

**Configurable.** Each regulator's rule set is its own file —
`rules/fsc_tw.yaml`, `rules/mas_sg.yaml`, `rules/sec_us.yaml`. Adding
a new jurisdiction is one new file, not a code change.

---

## 4. Brand safety

**Catches:** Creators with prior public scandals, doxxing controversies,
association with rug-pulls, or recent participation in coordinated
attacks against competitors. Also catches creators whose content style
is brand-incompatible (e.g. extreme political content where your client
is a B2B SaaS).

**Mechanism.** A combination of:
- Web search lookup for `[creator handle] + [scandal | fraud | rug-pull
  | doxxing | controversy]`
- Claude review of post tone, comment-section dynamics, and known
  community-warning signals
- A maintained "do not partner" denylist (per client)

**Configurable.** The brand-safety threshold is a slider per client;
some clients are stricter, some can absorb edgier creators.

---

## 5. Disclosure standards

**Catches:** Creators whose past paid promotions were not properly
disclosed under the relevant regulator's standards. (E.g., FTC
endorsement guides in the US, Taiwan FSC guidance for financial
products, EU influencer regulation.)

**Mechanism.** Claude reviews tagged "ad" / "promo" / "sponsored"
content in the last 90 days for compliance with the local disclosure
standard. Creators who routinely fail disclosure are flagged — they
will create regulatory exposure for the client too.

**Configurable.** Per-jurisdiction disclosure standard.

---

## 6. Conflict of interest

**Catches:** Creators currently in active partnership with a direct
competitor of the client. Sometimes acceptable (parallel partnerships
in non-exclusive markets), sometimes a deal-killer.

**Mechanism.** Active-partnership detection from referral codes in bio,
recent sponsored content tags, and cross-platform promotion patterns.
The output is a list of competitor brands the creator currently
promotes.

**Configurable.** The client decides per-competitor whether the conflict
is acceptable.

---

## What "pass" means

A pass means **all six checkpoints returned `passed=true`**. The
candidate's `funnel_state` advances to `COMPLIANT` and outreach can
fire.

A fail means the candidate's `funnel_state` becomes
`REJECTED:compliance` with the specific checkpoint and evidence
attached. Failed candidates are reviewable in the compliance log but
never receive outreach automatically.

If a borderline case warrants human review, the orchestrator can route
to a Slack channel for ad-hoc adjudication. The default policy is "fail
closed": if the gate isn't sure, the candidate doesn't pass.

---

## Why it's a gate, not a filter

The vocabulary matters. A *filter* removes things from a stream and
keeps the rest moving. A *gate* stops everything until the check is
done. KOL compliance has to be a gate.

If your pipeline filters compliance — i.e., it sends outreach in
parallel and removes the failures — you've already created the
regulatory exposure. The post is sent, the message is in the creator's
inbox, and Legal is now in cleanup mode.

The gate posture means you ship slightly slower (compliance review
adds 30–60 seconds per candidate) but you never ship a non-compliant
contact. This is the correct trade.

---

## Logging and audit trail

Every check, pass or fail, writes a row to `compliance_check`:

| candidate_id | checkpoint    | passed | reason            | evidence_url               | checked_at          |
| ------------ | ------------- | ------ | ----------------- | -------------------------- | ------------------- |
| x:traderjoe  | jurisdiction  | true   | TW audience 73%   | -                          | 2026-04-30 10:14:02 |
| x:traderjoe  | sanctions     | true   | not on SDN        | -                          | 2026-04-30 10:14:02 |
| x:traderjoe  | regulated     | false  | leverage advice   | https://x.com/.../1234567  | 2026-04-30 10:14:03 |
| x:traderjoe  | (gate result) | false  | regulated content | -                          | 2026-04-30 10:14:03 |

When Legal asks "why didn't we partner with X?" or "why DID we partner
with Y?", this table is the answer in 10 seconds.
