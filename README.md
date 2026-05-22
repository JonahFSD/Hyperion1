# Hyperion 

Authored by Jonah Elliott https://github.com/JonahFSD and Joe Comer https://github.com/Joeorisit 

I spent a few days trying to build an AI-powered equity research engine. The core idea was that if you run SEC filings through a large language model and extract structural fingerprints using Sparse Autoencoders, you can find companies that operate similarly, even across industries, better than the US government's classification system.

It works. Tested across 25 years of market data and 14.9 million company pairs, it beat SIC codes in every single time window we checked. The statistical significance was nearly double the gold standard threshold used in quantitative finance. 96–99% of the signal survived after removing all known risk factors. It's finding something real.

The problem wasn't the research, sadly, it was the market. Better company similarity is a vitamin, not a painkiller. Analysts already have ways of finding comps that are good enough. This is measurably better, but nobody's losing sleep over it. The thing I built is cool and works, it just solves a problem that isn't painful enough for anyone to pay to fix.

So this is a research finding, not a product. Here's all the code.

---

## The idea

Every public company files a 10-K (annual report) with the SEC describing how the business actually works. The government classifies these companies using SIC codes, which are four-digit industry labels from the 1930s. SIC codes tell you what a company *makes*. A cloud infrastructure company and a cybersecurity startup might share the same SIC code ("Prepackaged Software") despite being completely different businesses. Meanwhile, a payments processor and a specialty insurer might be in different SIC codes despite doing structurally identical things, like taking thin margins on high transaction volumes.

The question was: can we do better?

### What SAE fingerprints are

When a large language model reads a 10-K it builds an internal mathematical representation of the business. A Sparse Autoencoder (SAE) extracts the meaningful patterns from that representation, producing a 131,072-dimensional fingerprint per filing. Most values are zero. The non-zero ones correspond to specific structural features the SAE learned to recognize.

We reduce these to 4,000 dimensions with PCA (keeping ~90% of the information), then measure company similarity using cosine similarity.

The SAE is `EleutherAI/sae-llama-3-8b-32x` applied at layer 30 of Llama 3. The features come from [Molinari et al.'s HuggingFace dataset](https://huggingface.co/datasets/marco-molinari/company_reports_with_features) — roughly 27,888 filings spanning 25 years.

This builds on the ACL 2025 paper [*Interpretable Company Similarity with Sparse Autoencoders*](https://github.com/FlexCode29/company_similarity_sae) by Molinari, Shao, Ménard, and Music. Their code is in `company_similarity_sae/`. Everything in `experiments/` is our independent extension.

---

## What we found

We ran roughly 13 statistical tests across 3 layers of validation. All p-values are bootstrapped (BCa, ticker resamples, 10,000 iterations). Benjamini-Yekutieli FDR correction is applied to the 3 Layer-1 delta tests (`experiments/1a_07_verdict.py:30,521-538`); the Layer-2 retrieval tests (T01-T05) and the 1B residual tests are reported with raw bootstrap CIs and aren't folded into the joint family.

**SAE fingerprints beat SIC codes at grouping similar companies, with one important caveat.** We replicated the paper's Mean Correlation result (MC = 0.358981 vs paper 0.359, to 1e-3) on the same 14.9 M pairs and 25 years — `experiments/artifacts/1a_replication.json`. The SAE-SIC delta of +0.128 over the pooled corpus has BCa 95% CI [0.096, 0.186] and a bootstrap pseudo-z of 5.57 (delta over bootstrap standard deviation, ticker-resamples, no degrees of freedom — `1a_bootstrap.json`). Separately, the descriptive 5-year rolling-window count has SAE > SIC in 21 of 21 windows, with the caveat that consecutive windows overlap by 4 of 5 years (`1a_rolling.json`). The headline-level Phase-1 verdict file reads `CONDITIONAL`, with three open diagnostic flags (SAE-SIC and SAE-SBERT advantages growing over time, bootstrap z₀ = 0.634) — `1a_verdict.json`. The caveat: equal-weighted MC favours SAE partly because SAE's MST-with-threshold clustering produces many size-2 clusters; under pair-weighted MC the ranking inverts (SAE 0.151 vs SIC 0.252 vs SBERT 0.210, `1a_cluster_size_control.json`). The fair reading is that SAE is a high-precision, low-recall pair-finder — which is exactly the use case the within-industry re-ranker (below) targets.

**Within an industry, SAE adds real precision as a re-ranker.** Start with companies in the same 2-digit SIC and re-rank by SAE cosine. At K=1, the SAE-picked nearest neighbour has mean return correlation **0.263** vs **0.208** for a random same-SIC peer — a lift of **+0.055** absolute (or +26.6% relative), CI excludes zero at K=1,3,5,10 across all 25 years (`1a_11_t04_result.json`). Globally — across all industries at once — SAE's top-10 mean correlation **0.222** actually *underperforms* SIC's **0.244** (`1a_11_t03_result.json:sae_top10_vs_sic.sae_wins: false`). So the only fair architectural claim is the narrow one: SIC for candidate generation, SAE for within-industry re-ranking.

**The within-SIC signal isn't just known risk factors.** We stripped out all 5 Fama-French factors (market, size, value, profitability, investment) and re-ran the within-SIC re-ranking on the residuals. The K-lift survived at 95.8% (K=1) to 99.1% (K=10) — `1b_factor_adjustment_result.json`. Median FF5 R² on monthly returns is 3.0%, so factor models barely touch this signal. Two scope notes: the 96–99% figure is the survival of the within-SIC K-lift specifically, not of the headline MC = 0.359 (which 1B never re-runs on residuals); and the alignment of the 12-month return vector to calendar months is assumed (m₀ = January) rather than spot-checked against Yahoo Finance.

**But it doesn't predict returns.** We ran a production-realistic pairs trading backtest (walk-forward PCA, no look-ahead bias, 100-trial random-pair placebo) and an analog return prediction test. Both null. Structural similarity is real but doesn't translate to tradeable signal. More importantly the market for better company comps turned out to be a vitamin, not a painkiller. Analysts have workarounds that are good enough. This is better, but not urgently better.

---

## Reproducing the results

### Setup

Python 3.9+, about 4 GB of disk space for the HuggingFace data.

```bash
git clone https://github.com/YOUR_USERNAME/Hyperion.git
cd Hyperion
python3 -m venv hyperion-env
source hyperion-env/bin/activate   # Windows: hyperion-env\Scripts\activate
pip install -r requirements.txt
```

### The short version

Five scripts reproduce the core finding:

```bash
python experiments/1a_01_data.py                      # download & verify data
python experiments/1a_02_replicate.py                  # replicate MC = 0.359
python experiments/1a_06_rolling.py                    # SAE wins 21/21 windows
python experiments/1a_11_t04_within_sic_precision.py   # +26% within-SIC lift
python experiments/1b_factor_adjustment.py             # 96-99% survives FF5
```

To see the null backtest:

```bash
python experiments/2a_01_walkforward_pca_diagnostic.py
python experiments/2a_02_pair_universe.py
python experiments/2a_03_return_matrix.py
python experiments/2a_04_pairs_trading.py              # pairs trading (null)
python experiments/2b_analog_prediction.py             # analog prediction (null)
```

### Practical notes

First run downloads data from HuggingFace, takes a few minutes, then it's cached. No GPU required (everything runs on CPU). Most scripts finish in under 5 minutes. The pairs trading backtest and rolling window analysis can take 15–30 minutes. The streaming scripts handle the 15M-row similarity dataset without blowing up your RAM, but 8 GB is a comfortable minimum.

A small reproducibility wart: `experiments/1a_11_t04_within_sic_precision.py` uses `os.path.join(...)` without importing `os` at the top of the file, and reads a SIC mapping from `phase1_artifacts/idx_to_sic2.pkl` that no other committed script materialises. To re-run that script from a clean clone you need to add `import os` and rebuild the SIC mapping the way `experiments/1b_factor_adjustment.py:124-126` does. The committed result JSON in `experiments/artifacts/1a_11_t04_result.json` is from before this regression.

---

## What's in here

`company_similarity_sae/` is the upstream ACL paper code, byte-identical to upstream HEAD — do not edit. `experiments/` is our work. `experiments/legacy/` holds our earlier monolithic Phase-1 script (`run_phase1.py`, 753 lines), now superseded by the numbered modular scripts.

The experiment scripts run in order within each phase:

**Phase 1A** (`1a_01` through `1a_11`) — Replication and core statistical validation. Does the paper's result hold up? (Yes.) Is it robust across time? (Yes.) Is it sensitive to the clustering threshold? (Not really.)

**Phase 1B** (`1b_01` through `1b_03`) — Factor adjustment. Is the signal just size, value, or momentum in disguise? (No. 96–99% survives Fama-French 5-factor regression.)

**Phase 2A** (`2a_01` through `2a_04`) — Walk-forward pairs trading. Can you trade on this? (No.)

**Phase 2B** (`2b`) — Analog return prediction. Do historical structural analogs predict future returns? (No.)

**Phase 2C** (`2c_01`) — Event prediction. Do fingerprint changes predict corporate events? (Explored, inconclusive.)


---

## Data

Everything is public:

- [SAE features](https://huggingface.co/datasets/marco-molinari/company_reports_with_features) — ~27,888 filings, 131K-dim vectors
- [Company metadata + returns](https://huggingface.co/datasets/Mateusz1017/annual_reports_tokenized_llama3_logged_returns_no_null_returns_and_incomplete_descriptions_24k)
- [Pre-computed cosine similarities](https://huggingface.co/datasets/v1ctor10/cos_sim_4000pca_exp) — 15M rows, 645 MB
- [Fama-French 5 factors](https://mba.tuck.dartmouth.edu/pages/faculty/ken.french/ftp/F-F_Research_Data_5_Factors_2x3_CSV.zip)

---

## Attribution

This project builds on [*Interpretable Company Similarity with Sparse Autoencoders*](https://github.com/FlexCode29/company_similarity_sae) by Marco Molinari, Victor Shao, Brice Ménard, and Léopold Music (ACL 2025). Their code lives in `company_similarity_sae/` and is byte-identical to upstream HEAD (verified against `gh api repos/FlexCode29/company_similarity_sae/git/trees/HEAD`). Our contribution is the independent validation and backtest in `experiments/`.

**Note on our own legacy code, not the upstream paper's:** `experiments/legacy/run_phase1.py` is an early Hyperion-authored monolithic script (it lives in our tree, not in upstream — `gh api .../contents/run_phase1.py` returns HTTP 404 against FlexCode29). It computes pairwise return correlations by truncating to common array length without date alignment (line ~240: `r1[:min_len], r2[:min_len]`). If two companies have return series starting in different years, this correlates misaligned time periods. The numbered modular scripts under `experiments/` avoid this — they use the pre-computed correlation column from the HuggingFace dataset, not this function. The legacy script is kept for reference only.

## License

MIT
