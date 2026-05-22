# Left for the author

Things found during the documentation honesty pass that are out of scope for a docs-only change. Shortest-feedback-loop first.

## Code regressions worth fixing soon

1. **`experiments/1a_11_t04_within_sic_precision.py` is broken on a fresh clone.**
   - Line 17 calls `os.path.join(...)` but `import os` is missing from the import block at the top.
   - The path it constructs is `<repo>/phase1_artifacts/idx_to_sic2.pkl`, but no committed script materialises that pickle. The same SIC-to-2-digit mapping is built in-memory by `experiments/1b_factor_adjustment.py:124-126`; lifting that into a small standalone "build the SIC mapping" step (or having T04 build its own) would unbreak the script.
   - The committed result `experiments/artifacts/1a_11_t04_result.json` was produced before this regression and is still the source of the +0.055 K=1 lift quoted in the README. Anyone re-running the script will hit the bug.

## Methodology — flagged but not resolved

2. **FF5 month alignment is still assumed, not verified.** `experiments/1b_factor_adjustment.py:182` treats vector index 0 of `logged_monthly_returns_matrix` as calendar January of the focal year. `HANDOFF-1A.md` §5.2 explicitly lists this as an "UNVERIFIED CRITICAL ASSUMPTION" that needs spot-checking against Yahoo Finance for 3–5 ticker-years (especially fiscal-year-non-December companies). The README now flags this caveat; a one-time spot-check artifact would close it.

3. **Phase-1 walk-forward PCA.** The 4,000-dim PCA basis used for Phase 1 is fit globally on all 25 years (upstream). The 21 rolling windows regroup that globally-trained representation. Phase-2 already does proper walk-forward (`2a_01_walkforward_pca_diagnostic.py`); if the same walk-forward representation were used to recompute the Phase-1 rolling numbers, "21/21 windows" could be claimed as walk-forward without caveat.

4. **1C feature-shuffle permutation test.** Designed in `HANDOFF-1A.md` §3.5 and `investigation/tests/T08-permutation-test.md`, never implemented. The "algorithmic-artifact" null is therefore not formally ruled out. The within-SIC use case stands on T04 + 1B regardless, so this is academic completeness rather than product-gating.

5. **Joint BHY across the full ~13-test family.** Currently applied to the 3 Layer-1 delta tests only. Extending to T01-T05 and the 1B residual tests is a one-script change.

## Internal consistency

6. **`HANDOFF-1A.md` §5.2 vs `docs/AutoShop-SAE-Validation-Appendix.md` §5.3.** The HANDOFF says the month-alignment spot-check is unverified. The AutoShop appendix (line 322-324) claims the check was performed and the assumption held. There's no spot-check artifact in `experiments/artifacts/`. One of these two documents is wrong about whether the check happened — worth resolving so future readers don't get contradictory answers.

7. **The "Note on the upstream code" / now "Note on our own legacy code" still names line ~240 of `run_phase1.py` for the truncation bug.** `HANDOFF-1A.md` §4 documents a *separate* bug at lines 531-538 (positional FF5 alignment). Both bugs are real and both are in `experiments/legacy/run_phase1.py`. If you cite the file in writing or interviews, mention both: line ~240 truncates correlation pairs without date alignment, and lines 531-538 do positional FF5 alignment (i.e., regress 2005 returns against 1963 factors). The numbered modular scripts fix both.

## Repo hygiene

8. **`phase1_artifacts/` lives at the repo root and contains the runtime artifacts the scripts expect**, but the committed result JSONs live in `experiments/artifacts/`. The active scripts work with one or the other depending on their hardcoded path. Worth picking a single convention and migrating; the simplest is `experiments/phase1_artifacts/` so script-relative paths resolve from the script directory.

9. **`run_phase1.py` references in the rest of the tree.** I updated the README's Attribution section; I did not search every other doc. Worth `grep -rn run_phase1` to catch any stragglers in `docs/research/`, `investigation/`, etc.

## Not done (would belong in a separate pass)

- A small `make reproduce` (or equivalent shell wrapper) that re-runs the JSON-consuming scripts against `experiments/artifacts/` and asserts the README numbers still match. This would catch any future drift between README and artifacts automatically.
- `investigation/STATUS.md` predates Phase 2; if you keep it, a quick refresh to record that 2a (pairs trading) and 2b (analog prediction) came back null would round out the honesty record.
