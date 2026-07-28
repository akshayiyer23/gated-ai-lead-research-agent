# Gated AI Lead-Research Agent

I built a lead-research pipeline that runs AI on a lead only after it passes a data-quality gate, and only when the AI can cite a real source. It verifies the email, matches the company, gates on both, researches with a browsing AI agent, forces a refusal when evidence is thin, and writes typed results into HubSpot.

On a 15-lead test set, the gate blocked AI research on 7 of 15 rows. That skipped 47% of the AI calls. The 8 rows it did research came back with zero fabricated fields.

## Why most AI research columns fail

Point an AI research column at a whole lead list and two things go wrong.

First, you pay to research junk. Dead emails and fake domains hit the AI anyway, and every one of them costs a credit.

Second, and worse: on a lead with no real signal, the AI makes something up. It invents a funding round, guesses a headcount, writes a "buying signal" that never happened, and files all of it in your CRM as fact. A rep reads it, trusts it, and acts on a lie. A research tool that fabricates with confidence does more damage than no tool at all.

I built this to fix the second problem first.

## How it works

```
Lead → verify email → enrich company → GATE → research (if passed) → refusal check → typed fields → HubSpot
                                          |
                                     fail → stop, spend nothing
```

**The gate needs two yeses.** A lead reaches the AI only if the email verifies as deliverable (ZeroBounce) AND the domain resolves to a real company with firmographic data. One signal is not enough. A real company with a dead contact email fails. A clean-looking email on a made-up domain fails. I check both because either one alone lets bad leads through.

**The AI runs behind the gate.** The browsing research agent (Claygent, Argon model) fires only on rows that passed. On the test set that was 8 of 15. The other 7 cost nothing.

**The agent has to refuse.** The prompt gives it one hard rule: if you cannot ground a claim in a page you actually opened, return `{"status": "insufficient_evidence"}` and stop. I handle that refusal as its own path, so a refusal never gets parsed as if it were a finding. The anti-hallucination guarantee lives in the prompt contract, not in wishful thinking.

**Results land as typed fields.** Each result splits into `summary`, `buying_signal`, `confidence`, and `sources`, and each writes to its own HubSpot property. A rep sees the claim, how confident the agent was, and the exact URLs it read. No paragraph of maybe-true text dumped into a notes box.

## What the run produced

| Metric | Value |
|---|---|
| Leads in | 15 |
| Passed the gate and got researched | 8 |
| Blocked by the gate, AI skipped | 7 (47% of AI calls avoided) |
| Fabricated or ungrounded rows | 0 |
| Confidence of researched rows | 7 high, 1 medium |
| Written to HubSpot | 8 contacts, 0 import errors |

Every researched row came back with a dated, sourced buying signal. Edera's $15M Series A with the announcement page. A live Greenhouse board standing in for a hiring signal. ElevenLabs' $500M Series D with the blog post that announced it. Specifics I can click, not filler.

## Stack

Clay ran the orchestration and the run-condition gating. ZeroBounce verified emails. A company-enrichment provider matched firmographics. Claygent (Argon) did the browsing research. HubSpot received the typed write-back.

## What breaks in production

The 15-row happy path is not the bar. Here is where this cracks at scale and what I would do about it.

Catch-all domains slip past a strict gate. ZeroBounce returns `catch-all` rather than `valid` for a lot of corporate mail servers, so a `valid`-only rule blocks real buyers. Production needs a catch-all policy: let them through with a flag, or add a second check.

Enrichment misses stealth companies. A brand-new or rebranded domain can fail the firmographic match even though the company is real. My gate errs toward blocking a good lead over admitting a bad one, but I would measure that miss rate instead of assuming it.

Sources rot. The agent cites pages it opened, and those 404 or move behind a login later. I would timestamp every source so a rep can tell fresh evidence from stale.

Cost climbs fast. Fifteen rows is free. A hundred thousand rows on a browsing model is not. I would add a per-lead budget and run a cheap model first, escalating to the strong one only on ambiguous rows.

Refusal needs tuning. Too strict and it refuses leads it could have researched. Too loose and hallucinations creep back in. That threshold deserves an eval set, not a gut call.

## What it demonstrates

Gating an AI column on data quality. Handling refusal as a first-class outcome. Carrying provenance with every claim. The pipeline spends less, never invents, and hands the CRM structured intelligence a rep can act on.
