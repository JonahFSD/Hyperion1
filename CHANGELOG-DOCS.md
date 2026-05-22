# Documentation rewrite — changelog

**Date:** 2026-05-22
**Scope:** documentation honesty pass on the active repo (`/Users/jonahelliott/Hyperion`). One file move; surgical edits to `README.md`. No analysis code, no result files, no experiment scripts were touched.

The active README was already significantly more honest than earlier versions (after the `46bc52e` "Refactor README to eliminate redundancy" commit and the post-Phase-2 "doesn't predict returns" framing). The remaining issues were narrow: the `run_phase1.py` mis-attribution, the t = 5.57 ↔ 21-window conflation, and three undisclosed limits (CONDITIONAL verdict, pair-weighted MC inversion, scope of the 96–99% FF5 survival number). All three are now stated alongside the numbers they qualify, in the same brief tone the rest of the README uses.

---

## Files changed

- `README.md` — four surgical edits inside "What we found", plus a small clarification in "Practical notes", an updated note in "What's in here", and a rewritten Attribution block.
- `company_similarity_sae/run_phase1.py` → `experiments/legacy/run_phase1.py` via `git mv`.

## File moved

- `company_similarity_sae/run_phase1.py` → `experiments/legacy/run_phase1.py`.
  - **Why:** the file is Hyperion-authored, not part of the ACL paper's upstream repo. Confirmed: `gh api repos/FlexCode29/company_similarity_sae/contents/run_phase1.py` → HTTP 404. Every other file in `company_similarity_sae/` is byte-identical to upstream HEAD. Leaving an author-written file inside the upstream-labelled directory was causing the README to describe a bug we introduced as "a note on the upstream code."

---

## Claims that changed in `README.md`

For each: what the README said before, what it says now, which artifact justifies the new wording.

### 1. "SAE fingerprints beat SIC codes at grouping similar companies."

| | Old | New | Artifact |
|---|---|---|---|
| Headline | "SAE fingerprints beat SIC codes at grouping similar companies." | "SAE fingerprints beat SIC codes at grouping similar companies, with one important caveat." | — |
| MC replication | "MC = 0.359" | "MC = 0.358981 vs paper 0.359, to 1e-3 … on the same 14.9 M pairs and 25 years" | `1a_replication.json` |
| t-stat framing | "SAE won in all 21. The t-statistic was 5.57 — finance research considers t > 3.0 the gold standard." | The two procedures are now reported separately and accurately: the bootstrap pseudo-z (5.57 = delta over bootstrap standard deviation on the pooled corpus, no degrees of freedom) and the descriptive rolling-window count (21/21, windows overlap by 4 of 5 years). The "t > 3.0 gold standard" comparison was removed because the bootstrap pseudo-z is not directly comparable to a Harvey-Liu-Zhu t-stat. | `1a_bootstrap.json:deltas.sae_minus_sic.t_stat`, `1a_rolling.json:sae_vs_sic` |
| Verdict | Not mentioned. | Added: "The headline-level Phase-1 verdict file reads `CONDITIONAL`, with three open diagnostic flags (SAE-SIC and SAE-SBERT advantages growing over time, bootstrap z₀ = 0.634)." | `1a_verdict.json:overall`, `1a_verdict.json:flags` |
| Pair-weighted inversion | Not mentioned. | Added: "The caveat: equal-weighted MC favours SAE partly because SAE's MST-with-threshold clustering produces many size-2 clusters; under pair-weighted MC the ranking inverts (SAE 0.151 vs SIC 0.252 vs SBERT 0.210). The fair reading is that SAE is a high-precision, low-recall pair-finder — which is exactly the use case the within-industry re-ranker targets." | `1a_cluster_size_control.json` |

### 2. "Within an industry, SAE finds 26% more genuinely similar companies."

| | Old | New | Artifact |
|---|---|---|---|
| Numbers | "precision improves by 26% (measured by return correlation)" | "At K=1, the SAE-picked nearest neighbour has mean return correlation **0.263** vs **0.208** for a random same-SIC peer — a lift of **+0.055** absolute (or +26.6% relative), CI excludes zero at K=1,3,5,10 across all 25 years." | `1a_11_t04_result.json` |
| Global retrieval | "Globally across all industries, SIC is actually the better first filter — so the natural architecture is SIC first, SAE second." | "Globally — across all industries at once — SAE's top-10 mean correlation **0.222** actually *underperforms* SIC's **0.244** (`sae_top10_vs_sic.sae_wins: false`). So the only fair architectural claim is the narrow one: SIC for candidate generation, SAE for within-industry re-ranking." | `1a_11_t03_result.json:sae_top10_vs_sic` |

### 3. "The signal isn't just known risk factors."

| | Old | New | Artifact |
|---|---|---|---|
| Headline | "The signal isn't just known risk factors." | "The within-SIC signal isn't just known risk factors." (scoped) | — |
| Survival number | "96–99% of the SAE signal remained" | "The K-lift survived at 95.8% (K=1) to 99.1% (K=10)" | `1b_factor_adjustment_result.json:t04_comparison.K_*.survival_ratio` |
| Scope | Not stated. | Added: "the 96–99% figure is the survival of the within-SIC K-lift specifically, not of the headline MC = 0.359 (which 1B never re-runs on residuals)" | `1b_factor_adjustment_result.json` |
| Month alignment | Not stated. | Added: "the alignment of the 12-month return vector to calendar months is assumed (m₀ = January) rather than spot-checked against Yahoo Finance — see `HANDOFF-1A.md` §5.2" | `HANDOFF-1A.md:288-297` |

### 4. "We ran 13 statistical tests…" line

| Old | New | Artifact |
|---|---|---|
| "Multiple testing is corrected with BHY false discovery rate." | "Benjamini-Yekutieli FDR correction is applied to the 3 Layer-1 delta tests (`1a_07_verdict.py:30,521-538`); the Layer-2 retrieval tests (T01-T05) and the 1B residual tests are reported with raw bootstrap CIs and aren't folded into the joint family." | `experiments/1a_07_verdict.py:30,521-538` |

### 5. Practical-notes block

Added a single sentence flagging the `experiments/1a_11_t04_within_sic_precision.py` regression (`os.path.join` without `import os`, plus a SIC-mapping pickle no committed script materialises). The committed JSON result is from before the regression.

### 6. "What's in here"

| Old | New |
|---|---|
| "`company_similarity_sae/` is the upstream ACL paper code, unmodified." | "`company_similarity_sae/` is the upstream ACL paper code, byte-identical to upstream HEAD — do not edit. … `experiments/legacy/` holds our earlier monolithic Phase-1 script (`run_phase1.py`, 753 lines), now superseded by the numbered modular scripts." |

### 7. Attribution

| Old | New |
|---|---|
| "**Note on the upstream code:** `run_phase1.py` computes pairwise return correlations by truncating to common array length without date alignment …" | "**Note on our own legacy code, not the upstream paper's:** `experiments/legacy/run_phase1.py` is an early Hyperion-authored monolithic script (it lives in our tree, not in upstream — `gh api .../contents/run_phase1.py` returns HTTP 404 against FlexCode29). It computes pairwise return correlations by truncating to common array length without date alignment …" |

---

## What was preserved

- The "vitamin not painkiller" framing and the "But it doesn't predict returns" bullet — those are the most honest parts of the active README and weren't touched.
- The MC = 0.359 replication, the +26% within-SIC lift, the 95.8–99.1% FF5 survival, the 14.9 M pairs, the 25 years, the median FF5 R² = 3%, the null Phase-2 pairs trading and analog-prediction results.
- The overall short, conversational tone — the surgical edits preserve sentence-level density and don't bloat the document.
- `company_similarity_sae/`, every script in `experiments/`, every JSON in `experiments/artifacts/`.
