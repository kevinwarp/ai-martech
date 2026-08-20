---
name: applovin-cohort-review
description: Compare the newest weekly cohort of AppLovin creatives against the existing creative library, split by agency vs brand source and by Audience Strategy, and classify every new creative as scale / remix / stop. Use when someone asks how the latest AppLovin creative cohort or batch is performing, wants new vs existing creative comparison, asks which new creatives to scale, remix, or kill, wants a weekly cohort readout, or says "how did this week's creatives do". Produces a chat-ready summary, not the full analytical HTML report.
---

# AppLovin Cohort Review

## Objective

Produce a concise, actionable summary answering one question:

> **Are our newest creatives outperforming the existing creative library, and what should we do next?**

Optimize for speed, readability, and decisions that feed the next production batch. The detailed HTML report is a separate artifact; this skill is the chat readout.

## Inputs

### Required
- AppLovin creative performance data (per creative_set, per day)
- Creative launch dates (see Cohort Definition for how these are derived)
- Audience Strategy and Campaign Type per creative
- Spend, revenue, conversion counts

### Recommended metadata
- Creative Name, Creative ID, Source (agency vs brand/other)
- Product, Persona, Creative Concept, Hook
- Creator / Format, Duration, Script or narrative beat
- Campaign, Campaign Goal

### Pulling the data (AppLovin reporting API)

```
GET https://r.applovin.com/report
  ?api_key=$APPLOVIN_REPORTING_API_KEY&format=json&report_type=advertiser
  &start=YYYY-MM-DD&end=YYYY-MM-DD&day_column=day
  &columns=day,creative_set,creative_set_id,campaign,impressions,clicks,ctr,cost,
           roas_0d,nc_d0_roas,nc_percent_d0_checkouts,nc_d0_checkout_rev,chka_usd_0d
  &limit=500&offset=0
```

Rules that matter:
- `roas_0d` and `nc_d0_roas` are returned as **percent**. Divide by 100 for the × multiple.
- `cost` is spend. `chka_usd_0d` is all-customer D0 revenue, `nc_d0_checkout_rev` is new-customer D0 revenue.
- History is capped at roughly **78 days**. An earlier `start` returns HTTP 400 "Start date is too far in the past".
- Pull `campaign` as a dimension so each creative can be mapped to an Audience Strategy. A creative_set running in more than one campaign must be reported per strategy, never blended across strategies.
- Paginate with `offset` in multiples of `limit` (max 500).
- Never hardcode the API key. Read it from the environment or a secret store.

## Cohort Definition

### Launch date
A creative's launch is the **first calendar day it served an impression**. There is no creation-date field, so derive it as `min(day) where impressions > 0`. Its cohort is the **ISO week** of that day.

### Recent Cohort
The most recent ISO-week cohort where creatives:
- have accumulated at least **7 complete days** of delivery, and
- clear the confidence threshold below

If the newest week is not yet 7 days mature, report it as **Too Early** and run the comparison on the last mature cohort instead. Never silently skip it.

### Existing Creative Library
All creatives that launched before the Recent Cohort and received meaningful delivery **during the same comparison window**.

Exclude:
- paused creatives with no delivery in the window
- creatives below the confidence threshold
- creatives launched inside the Recent Cohort

### Confidence threshold (defaults, tune per brand)
Include a creative when **spend ≥ $250** OR **D0 checkouts ≥ 8** in the window. Anything below is `⚪ Insufficient Data`.

### Censored cohort
Creatives already live when the API history window opens have a launch date that cannot be recovered. Bucket them as **"Pre-window / established"** and never assign them a false launch week. Say so in the readout when they carry material spend.

## Comparison Methodology

- Always compare over the **same calendar window**. Never compare lifetime vs recent, or across different seasonal periods.
- Benchmark each creative against **its own Audience Strategy**. Never compare Discovery against Prospecting.
- Map Audience Strategy from the campaign name (`_Discovery`, `_Prospecting`, `_Universal`, `_Testing`). If a campaign name carries no audience token, report it as `Unmapped` and flag that the naming convention needs the audience type added.

### Required segmentation

Always report these three cuts, in this order:

1. **Overall**: Recent Cohort vs Existing Library
2. **By source**: agency-delivered creatives vs brand/other creatives, within the same weeks
3. **By Audience Strategy**: Discovery, Prospecting, Universal, plus any other strategy the advertiser runs

Show only rows that have data. Suppress empty segments rather than printing zeros.

## Metrics

**Primary**: D0 ROAS · NC-D0 ROAS · new-customer % of D0 revenue · CPA · CTR · CVR
**Secondary**: matured D7 ROAS (only when fully matured) · AOV · spend · revenue

When explaining why a creative won or lost, read the metrics as signals:

| Signal | What it points to |
|---|---|
| CTR | Hook quality, thumbnail, first 3 seconds |
| CVR | Messaging, product proposition, landing-page match |
| CPA | Acquisition efficiency |
| new-customer % | Whether it is doing net-new acquisition or re-selling existing buyers |
| D0 ROAS | Overall creative effectiveness |

### Benchmarks
Set these per brand at the start of the engagement. Reference defaults:
- New-customer D0 ROAS **≥ 2.0×** is the scale trigger
- Blended target **≥ 3.25×**, North Star 4× net sales
- AppLovin D0 is in-platform and directional. The brand's own attribution platform (for example Triple Whale linear-paid) is the **source of truth** for spend decisions. State this whenever a recommendation implies a budget move.

## Output Format

### 1. Executive Summary
Two to three sentences covering: whether the cohort outperformed or underperformed, the biggest driver of that result, and the single most important creative observation. Explain what changed. Do not restate metrics.

> **Example tone:** The W32 cohort came in 6% above the existing library on D0 ROAS, driven by conversion rate rather than clicks. Agency creatives carried the week at 1.58× against the brand library's 1.25× in the same days, with Problem→Solution the strongest concept in the batch.

### 2. Cohort vs Existing Library

One table. Δ is always **new vs existing, same dates**.

| Segment | New | Existing | D0 ROAS Δ | NC-D0 Δ | CPA Δ | CTR Δ | CVR Δ | Assessment |
|---|---|---|---|---|---|---|---|---|

Rows: Overall, then each source, then each Audience Strategy with data.

Assessment values: 🟢 Improving · 🟡 In Line · 🔴 Declining

### 3. Creative Classification

Every qualified Recent Cohort creative appears **exactly once**, in one of four buckets. Use the identical field template for all four:

**Creative · Audience Strategy · Source · Concept · Format · Primary KPI · Why · Action**

**🟢 Emerging Winner** — materially above the Audience Strategy benchmark.
Action: scale, build variants, remix into adjacent concepts.

> **Example**
> Creative A · Discovery · Agency · Problem→Solution · UGC · +38% D0 ROAS
> **Why:** high CTR combined with a materially higher CVR means both the hook and the product messaging land with net-new users. Early product demonstration establishes value fast.
> **Action:** scale and produce three variants holding the first 5 seconds.

**🟡 In-Line Performer** — roughly at the strategy baseline.
Action: change the one component the metrics implicate, hold the rest.

> **Example**
> Creative B · Prospecting · Brand · Testimonial · Creator · +2% D0 ROAS
> **Why:** CTR beats benchmark but CVR is flat, so the hook works while the value proposition is not differentiated.
> **Action:** remix the middle third with a product-proof beat, keep the opening.

**🔴 Underperformer** — materially below the strategy benchmark.
Never write "low ROAS". Name a probable cause: weak hook, offer introduced too early, thin product explanation, no social proof, hook-to-landing-page mismatch, assumes product familiarity, weak differentiation, slow setup.
Action must be specific, for example "replace the opening 5 seconds and keep the remainder" or "reposition around a different customer problem".

**⚪ Insufficient Data** — below the confidence threshold.
Include creative name, current spend, days live, reason, and the recommended action (usually: keep collecting, revisit at 7 full days).

### 4. Final Takeaways

Exactly three sections, written about **patterns**, not individual creatives.

**What to Scale** — the attributes shared across Emerging Winners (Product Demo, Problem→Solution, Before & After, UGC, and so on).

**What to Remix** — creatives with partial success (high CTR / low CVR, or strong CVR / weak CTR), naming the component to change: hook, creator, opening visual, product proof, CTA, end card.

**What to Stop or Reposition** — recurring characteristics among Underperformers (offer-first Discovery creative, weak differentiation, slow intros, founder monologues, generic testimonials), closing with one concrete recommendation for the next batch.

## Writing Style

Write like a senior performance marketing strategist: concise, analytical, actionable. Avoid generic statements. Never simply restate a metric. Every recommendation states what happened, why it likely happened, and what to do next.

## Guiding Principles

1. Explain **why** performance changed, not just whether it changed.
2. Benchmark every creative against its own Audience Strategy.
3. Read CTR, CVR, CPA, NC-D0 and D0 ROAS **together** to infer creative strengths and weaknesses.
4. Separate agency from brand creatives in every weekly cohort, so agency contribution is always legible.
5. Flag data-quality gaps (unmapped audience strategy, censored launch dates, immature cohorts) rather than papering over them.
6. Prioritize recommendations that can directly drive the next round of production, remixing, and testing.

## Notes

If the failure-cause library or worked examples grow, move them to `references/creative-diagnosis.md` and link from here to keep this file fast to load.
