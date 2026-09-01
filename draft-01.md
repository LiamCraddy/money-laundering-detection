# Detecting Money Laundering in Transaction Networks

* A graph-based machine learning aproach on the IBM AML synthetic dataset*

## 1. Problem

Money laundering is the process of diguising the origin of illegal obtained funds by moving them through a series of financial transactions. Trying to Automate this problem is hard for two reasons: laundering is rare (roughly 1 in 1,000 transactions in this data), and criminals deliberately structure their activitely to blend in with legimitate transactions. The cost of getting it wrong cuts both way, false negative let crime through, false positives bury analysts in useless alerts. This is a billion dollar issue one which will no be discovered in this but instead understand why.

This project asks: **can laundering accountss be distinguished from legitimate ones using machine leaerning, and how much does network structure matter

## 2. Data
IBM "Transactions for Anti-Money Laundering" synthetic dataset, **HI-Small**
variant (higher illicit ratio, small size).

- ~5 million transactions over ~10 days
- Each transaction: sender account, receiver account, amount, currency, format,
  timestamp, and a binary `Is Laundering` label
- A companion patterns file labels laundering transactions by which of 8 named
  structures they belong to (cycle, fan-in, fan-out, scatter-gather, etc.)


*Why synthetic:* real transaction data is restricted for privacy and proprietary
reasons, and reliable labels are nearly impossible to obtain. Synthetic data with
ground-truth labels makes supervised learning tractable.

**Known caveat (from dataset docs):** some laundering transactions fall just after
the stated date range — a potential source of temporal leakage to guard against.

## 3. Exploratory findings
### 3.1 The imbalance defines the problem

Laundering rate 1 in 981 transactions.
So accuracy is a useless metric (predicting "all clean" score ~99.9%).
So this Evaluation focuses on **precision** and **recall** for the laundering class.

### 3.2 No single feature separates the classes
- Median transaction amount: laundering ≈ 8,667 vs legitimate ≈ 1411. Laundering Moves larger sums but no so large that a simple threshold works. Since many legitimate transactions are also large.
- Distribution plots of per account features (e.g. No. of distinct receivers) show laundering and legitimate accounts heavily overlapping, no separated
* Conclusion: Sounds obvious but Laundering is only separable via combinations of features which motivates a machine learning approach over hand written rules.*

### 3.3 Laundering patterns by eye
Looking through the data through individual suspect account revealed:
- A **cycle**: Money leaving an account and returning through intermediaries.
- A **fan-out**: one account distributing to many.

### 3.4 Pattern distribution (from the pattern file)
Counting all labelled laundering attempts by type discovered.
| Pattern | Attempts | Transactions |
|---|---|---|
| Cycle | 54 | 287 |
| Gather-scatter | 51 | 716 |
| Bipartite | 49 | 263 |
| Fan-out | 48 | 342 |
| Scatter-gather | 44 | 626 |
| Stack | 43 | 466 |
| Random | 41 | 191 |
| Fan-in | 40 | 318 |

*Key insight: Laundering is spread evenly across all 8 pattern with no single dominant shape to target. With most patterns reducing to two primitives **branching** (fans) and **looping** (cycles).*

## 4. Feature engineering

### 4.1 Account-level features
Per-account aggregates: transaction counts, distinct-partner counts, total
volumes, sender/receiver ratio, throughput, fan imbalance, pass-through score.
 
- The **pass-through score** (min of in/out partner counts) was designed to catch
  the "money floods in from many, straight out to many" structure seen in the
  scatter plot. Median doubled for laundering accounts (1 → 2).
- The **in/out volume ratio** showed *no* separation (1.00 vs 1.01) and was dropped
  — a deliberately documented negative result.

 ### 4.2 A negative result: direct reciprocity
A feature measuring whether an account's direct partners pay it back was built to
catch cycles. It **failed** (means 0.51 vs 0.47, no separation). Reason: the
observed cycles were multi-hop (A→B→C→A), so the return arrives from an account the
suspect never paid directly. Direct reciprocity looks for the wrong-sized loop.

### 4.3 Graph features
Building the full transaction graph enabled structural features:
- **on_cycle** (account sits on a strongly-connected component): laundering
  accounts are on a cycle **25%** of the time vs **4%** for clean — a ~6× enrichment.
- **PageRank** (network centrality).
- True graph in/out degree (later dropped — redundant with existing counts, scored
  0 importance).


## 5. Modelling
 
### 5.1 Baselines and the threshold lesson
| Model | Notes |
|---|---|
| Random Forest (default 0.5 threshold) | recall 0.015 — far too cautious, almost never flags |
| Random Forest (threshold sweep) | recall rises to ~0.32 at threshold 0.05, precision falls to ~0.06 |
| LightGBM | recall 0.74 at default threshold, but precision collapses to ~0.04 |


*Lesson: the decision threshold is as important as the model. Random Forest was
too quiet; LightGBM too loud. The precision/recall tradeoff is the central design
choice, not an afterthought.*

### 5.2 Does graph structure help? (random split)
- Account features only: **average precision 0.084**
- Account + graph features: **average precision 0.101** (~20% relative gain)
- *Confirms network structure carries signal that account-level features miss.*
- Feature importance: **PageRank ranked #1** overall; on_cycle ranked low (strong
  signal but only for the ~25% of laundering that is cyclic, so used rarely).

### 5.3 The honest evaluation (time-based split)
Switching from a random split to a realistic **train-on-past, predict-future**
split (as the dataset docs recommend):
- Average precision dropped from **0.101 → 0.043** (roughly halved).
- *This is the project's most important result.* Much of the apparent performance
  under random splitting was temporal leakage. On the realistic task — catching
  *novel future* laundering — the problem is genuinely hard. (Still ~5× better than
  random chance of ~0.008.)

## 6. What this shows
- Account-level features hit a clear ceiling (L-shaped precision-recall curve).
- Hand-built graph features measurably help but capture only a fraction of the
  network's structure.
- Realistic temporal evaluation is far harder than naive random evaluation — a
  distinction many analyses miss.


## 7. Next steps
- **Graph Neural Network:** learn from network structure directly rather than via
  a handful of hand-chosen summaries. The modest gain from manual graph features is
  the motivation — there is structure left on the table.
- **Hyperparameter tuning** using the held-out validation set (untouched so far).
- **Cycle-specific features** capturing multi-hop loops more fully.
- **System layer:** wrap the scored model in an analyst-facing dashboard showing
  flagged accounts and the local network explaining *why* each was flagged.


