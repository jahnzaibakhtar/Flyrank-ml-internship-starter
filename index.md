---
title: "Refresh Priority Scoring: A Ranked Action Engine Built on Real Search-Console Data"
---

# Refresh Priority Scoring: A Ranked Action Engine Built on Real Search-Console Data

## Abstract

Content teams have limited hours and thousands of pages — the operative decision isn't "is this
page declining," it's "which page does an editor open next." This project builds a ranked
content-review queue for the **Refresh / Content Opportunity Scoring** lane, using March 2026 of
FlyRank's warehouse release (9,841,378 client-content-day rows, aggregated to 331,437
client-content items). A transparent, hand-readable rule flags content that is visible (high
search impressions) but underperforming CTR relative to similarly-ranked peers, flagging 1,856
of 175,304 position-scoreable items. Along the way, the project surfaced three genuine
data-quality findings that shaped the final method: 47% of content has no real search-position
data at all, a naive median fill would have silently corrupted the position-band analysis, and
the warehouse table's two GA4-availability flags disagree for a meaningful share of clients. This
paper reports a rule-based baseline, not a trained model — the warehouse table has no pre-built
decline label, and inventing one without a real future outcome window would be exactly the
leakage trap this project was built to avoid.

## Introduction / Problem statement

**Decision:** which content items should an editor review for a refresh this week, out of
everything they could review.

**Who acts, and how:** a content editor with limited hours works down a ranked queue, starting
at the top.

**Cost of a wrong call:** a false positive wastes editor hours — the cheaper mistake. A false
negative lets a page keep losing visibility or engagement silently until the next review cycle —
the more expensive mistake. Because the costs are asymmetric, this project optimizes for the top
of the ranking, not overall coverage.

**Why not a plain rule alone:** even the "plain rule" used here required real investigation to
get right — position bands, CTR distributions, and availability flags all needed verification
against the actual data before a threshold could be trusted (see Data and Methodology below).
A rule written blind, without checking the schema first, would have silently used columns that
don't exist in this table or misread a sentinel value as a real measurement.

## Data

**Release used:** FlyRank internship warehouse, `fact_content_daily_performance`, partition
`month=2026-03` — a mid-panel month, never the sealed final-month `_sample` (June 2026), which
is reserved as the outcome window for any future past→future label.

**Grain, verified:** one row = one `report_date` × `client_hash_id` × `content_hash_id`.
Confirmed with a `GROUP BY` + `HAVING COUNT(*) > 1` probe: **0 duplicate groups** out of the
full partition.

**Row count and window, verified:** 9,841,378 rows, dates spanning 2026-03-01 to 2026-03-31
exactly — fully bounded to the intended month.

**Schema note:** the real column names differ meaningfully from the starter CSV used earlier in
this project (`client_hash_id`/`content_hash_id` instead of `client_id`/`content_id`, no direct
`ctr` or `ai_traffic_pct` columns — both had to be computed from raw counts, no `scroll_rate`
column — engagement instead comes from `ga4_engaged_sessions` and `ga4_total_engagement_sec`).
Every feature in this paper was rebuilt against the verified real schema, not assumed from the
smaller starter dataset.

**Excluded, and why:**
- `fact_content_daily_performance_sample` — the sealed final month.
- `fact_content_query_90d` — excluded from this slice; its per-content context columns repeat
  per row (require `ANY_VALUE`, never `SUM`) and its 90-day window overlaps the panel's final
  months.
- GA4-derived columns on rows failing the availability filter (see below) — zero-filled or
  absent, not genuine zero-engagement.
- No client names, domains, URLs, private queries, credentials, or raw exports appear anywhere
  in this paper or its repo.

**Known data-quality finding (worth stating plainly):** only 4.2% of March rows
(413,966 / 9,841,378) have a verified `ga4_data_available == True`. The remainder splits
between rows with no GA4 relationship at all (`NaN`, 3,018,741 rows) and rows where GA4 exists
for the client but this row predates their data start (`False`, 6,408,671 rows). Additionally,
the table carries a second, client-level flag (`client_has_ga4`) that disagrees with the
row-level flag for a meaningful share of clients — of 20 clients marked `client_has_ga4 = False`,
only about half are `NaN` across 100% of their rows as the flag's name would imply; the rest show
real `True`/`False` values on 5-100% of their rows. This paper treats the row-level
`ga4_data_available` flag as authoritative and does not rely on `client_has_ga4` beyond a rough
sanity check.

**gsc_avg_position missingness:** the sentinel-zero pattern (0 meaning "no data," not rank 0)
appears at a modest and fairly evenly distributed rate — the five highest-missingness clients
each sit in the 4.0-4.8% range, not concentrated in one outlier client.

## Methodology

**Assumptions:** the review queue only needs to be right at the top, so ranking quality (not
overall coverage) is the goal.

**Aggregation:** March's content-day rows were rolled up to one row per client-content pair —
summed clicks and impressions (for a computed CTR), mean position on GSC-available rows only,
summed engagement signals, and a computed AI-session share.

**Baseline rule (plain words):** flag content that is visible (impressions at or above the
month's median) **and** underperforms CTR relative to other content at a similar position band.
No fitted weights — the rule is one sentence.

**Signal checks performed before trusting the rule:**

*Signal 1 — GA4 availability vs. engagement:* median engaged sessions rise from 0 (low
availability bucket) to 0 (mid) to 1 (high availability bucket) — CONFIRMED in direction but
weak in magnitude: availability does track with engagement, but the effect is small across two
of three buckets, consistent with how sparse real GA4 coverage is this month (see Data).

*Signal 2 — CTR vs. position band:* initial analysis showed a suspiciously large 4-10 band
(237,752 of 331,437 items). Investigation found this was an artifact: 156,133 items (47% of all
content) have no real search-position data, and a naive fill with the overall median position
(9.0) placed every one of them inside the 4-10 band, more than doubling its true size. After
excluding items with no real position data, the clean, honest comparison is:

| Position band | n (real data only) | Median CTR |
|---|---|---|
| top3 | 13,136 | 0.00096 |
| 4-10 | 81,619 | 0.0 |
| 11-20 | 32,548 | 0.0 |
| 20+ | 48,001 | 0.0 |

CONFIRMED that position matters, but with an important caveat: median CTR is exactly 0.0 for
every band except top3. This isn't noise — over half of March's position-scoreable content
received zero recorded clicks despite having impressions. The CTR-gap signal can only
meaningfully discriminate within the top-3 band; elsewhere, "underperforming CTR" is
unmeasurable because there's no click signal to compare against at all.

**What this paper does not attempt:** a trained supervised model. The warehouse table has no
pre-built decline label — that field exists only in the smaller starter CSV and does not
generalize to this table's grain or schema. Building a genuine label requires a real
future-outcome window (e.g. comparing April performance against March), which this month-one
slice does not yet include. Inventing a proxy label without that future window would reconstruct
exactly the leakage pattern this project's earlier data-contract work was designed to catch.

## Results

**Baseline queue:** 1,856 of 175,304 position-scoreable content items were flagged
`visible_but_ctr_below_position_peers` (the remaining 156,133 of 331,437 total items lack real
position data and are excluded from this signal entirely — see Data).

**Top-10 review:** every item in the top 20 shares an identical score (0.00096). This is a
structural limit of the rule, not a bug: within the top3 band, every flagged item has
`ctr = 0.0`, so the gap formula assigns them all the same value — the displayed order is a tie-
break artifact, not a meaningful priority sequence among the flagged set. One client
(`client_73cda7b4e4f265ea`) appears 6 of 20 times in the top rows; that same client had the
highest position-data missingness (4.8%) found during the data-contract phase of this project,
worth investigating separately as a client-size effect versus a genuine elevated problem rate.
A likely false-positive pattern: a top-3-position item with 0 clicks could be brand-new content
that hasn't accumulated clicks yet, not declining content — position and impressions alone can't
distinguish "new and slow to start" from "established and failing."

## Limitations & honest framing

- This paper reports **observed, directional, decision-support** results only — no causal claim
  about refreshing content, and no claim about how a search engine ranks anything.
- No trained model is presented. The rule-based baseline is the full deliverable at this stage,
  by design, because a trustworthy trained label doesn't yet exist in this data slice.
- GA4-based signals in this analysis rest on only 4.2% of March's rows — any GA4-derived feature
  should be read as a weak, sparse signal, not a reliable one.
- The two GA4 availability flags in the source data disagree for a subset of clients in a way
  this analysis could not resolve; results should not be over-interpreted for the affected
  clients.
- 47% of content items (156,133 of 331,437) have no real search-position data and are excluded
  from the CTR-vs-position signal entirely — a much larger gap than the starter CSV's ~4%
  sentinel rate, and a reminder that this warehouse slice cannot speak for content lacking
  Search Console coverage at all.
- The baseline's single signal cannot rank flagged items against each other — every top-band
  item with zero clicks receives an identical score, so this method separates "worth a look"
  from "not," but not "most urgent" from "merely worth a look."
- History and data availability are uneven across the client base more broadly — this month's
  slice is internally well-bounded, but findings should not be assumed to generalize evenly
  across all 104 clients in the release.

## Ranked recommendations (action playbook)

1. Review the 1,856 items flagged `visible_but_ctr_below_position_peers` first — these are
   visible enough to matter and have a verified, position-adjusted CTR gap — but review the
   top3-band, zero-click items with extra scrutiny for the "brand-new content" false-positive
   pattern named above before assuming decline.
2. Add a tiebreaker (e.g. raw impression volume, or age of content if available) in the next
   iteration — the current single-signal score cannot distinguish urgency within the flagged set.
3. Treat GA4-based signals as supplementary, not primary, given the 4.2% real-availability rate
   this month — a low-engagement reading on a GA4-unavailable row is not evidence of anything.
4. Before building a trained model in a future iteration, construct the label from a genuine
   future month (e.g. April) rather than reusing any pre-built label field, since none exists at
   this grain.

## Reproducibility

- Repo: (https://github.com/jahnzaibakhtar/Flyrank-ml-internship-starter)
- Notebooks: `work/notebooks/w01_research_question.ipynb` through `w06_capstone.ipynb`
- All aggregation and scoring logic uses `pandas`; random elements (none in this baseline-only
  version) would use `random_state=42` if a model is added later.
- Metrics/queue outputs: `work/outputs/baseline_action_score.csv` (excluded from git per the
  project's leak-guard; regenerated on each run), plus any `work/outputs/*.json` receipts.

## Acknowledgments & data credit

Built on the FlyRank ML Internship dataset — [flyrank.ai](https://flyrank.ai).
