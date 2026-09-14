# Detecting Money Laundering in Transaction Networks

A graph-based machine learning approach to flagging laundering accounts, 
built on the IBM AML synthetic transaction dataset.

## Overview

Money laundering detection is hard for two reasons: it's rare (~1 in 1,000 
transactions here) and deliberately disguised to look legitimate. This project 
asks whether laundering accounts can be distinguished from legitimate ones 
using machine learning, and specifically how much network structure (as 
opposed to individual account behaviour) contributes.

Key result: hand-built graph features give a real but modest boost over 
account-level features alone, and moving from a random train/test split to 
a realistic time-based split roughly halves apparent performance, exposing 
how much of the "easy" signal was temporal leakage.

## Data

- IBM "Transactions for Anti-Money Laundering" synthetic dataset (HI-Small variant)
- ~5 million transactions over ~10 days, labelled by 8 known laundering patterns 
  (cycles, fan-in/out, scatter-gather, etc.)
- Synthetic because real transaction data is privacy-restricted and reliable 
  labels are practically unobtainable

  ## Approach

- **Account-level features**: transaction counts, distinct-partner counts, 
  volumes, pass-through score, fan imbalance
- **Graph features**: cycle membership, PageRank, degree — built from the 
  full transaction graph
- **Models**: Random Forest and LightGBM, evaluated on precision/recall 
  (accuracy is meaningless at this class imbalance)
- Two evaluation regimes compared: random split vs. realistic time-based split

## Key Findings

- No single feature separates laundering from legitimate accounts -
  separation only emerges from feature combinations
- Graph features improved average precision from 0.084 → 0.101 (~20% relative 
  gain) over account features alone; PageRank was the top-ranked feature
- Under a realistic time-based split, average precision dropped to 0.043 — 
  the project's central finding, showing most gains under random splitting 
  were leakage rather than real signal
- Several negative results are documented deliberately (e.g. in/out volume 
  ratio, direct reciprocity) - worth reading for what *didn't* work and why

## Structure

- `final_report.pdf` - full write-up: EDA, feature engineering, modelling, 
  and evaluation in detail
- `01_explore.ipynb` / relevant scripts — analysis and model code
- `02_explore.ipynb` / Engineering model
- `data/` - dataset or references to it

## Next Steps

Graph neural networks (to learn structure directly rather than via hand-built 
summaries), further hyperparameter tuning, multi-hop cycle features, and an 
analyst-facing dashboard for flagged accounts.

## Full Report
See [`final_report.md`](./final_report.md) for full methodology, exploratory 
analysis, and detailed results.










