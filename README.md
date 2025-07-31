# Blockhouse Task 2: Quantifying Slippage & Designing an Execution Algorithm

## Introduction & Slippage Mechanics  
In modern electronic markets, trading runs on a **limit‐order book** (LOB). Participants submit **limit orders** at specified prices, which rest in the book, and **market orders** that execute immediately against the best resting liquidity. A 10‐level LOB snapshot (MBP-10) captures the top ten bid and ask prices and sizes at time *t*.  

When you send a market order of **X** shares, it “eats” through these levels, causing your average execution price to deviate from the mid‐quote. We define the mid‐price:
m_t = (bid_{t,0} + ask_{t,0}) / 2    # unit: quote‐currency per share

and the temporary impact (per‐share slippage) for a buy is:
g_t(X) = (1 / X) * Σ_{i=1..k} [ (p_i - m_t) * q_i ]

For sells, use (m_t - p_i) instead. This exactly matches Figures 1–4 in the assignment prompt.

### Toy Example (with bid side in table too)

| Level | Ask Price pᵢ | Ask Size qᵢ | Bid Price pᵢ | Bid Size qᵢ |
|-------|--------------|-------------|--------------|-------------|
| 1     | $10.00       | 50 shares   | $9.95        | 75 shares   |
| 2     | $10.05       | 75 shares   | $9.90        | 50 shares   |
| 3     | $10.10       | 100 shares  | $9.85        | 100 shares  |


Mid‐price = (10.00 + 9.95)/2 = 9.975
gₜ(100) = (50·0.025 + 50·0.075) / 100 = $0.05/share
gₜ(200) would also clear 50@10.10, raising the average.

## Overview

This repository demonstrates how to:

1. **Compute “temporary impact”**—the per-share slippage incurred when a market order “eats” through multiple levels of a 10-level limit-order book.
2. **Model the shape of that impact (Q1)** using two parsimonious functional forms (linear and power-law), fitted on three tickers (SOUN, CRWV, FROG).
3. **Formulate and solve an optimal execution problem (Q2)** that allocates a fixed total volume across discrete decision times to minimize aggregate slippage.

---

## 1. Data & Preprocessing

- **Source:** MBP-10 snapshots from Databento (10 price levels of depth).  
- **Tickers:**  
  - **SOUN:** 231 965 snapshots  
  - **CRWV:** 104 640 snapshots  
  - **FROG:**  45 789 snapshots  
- **Key fields:**  
  - `bid_px_00…bid_px_09` and `ask_px_00…ask_px_09` (top 10 bid/ask prices)  
  - `bid_sz_00…bid_sz_09` and `ask_sz_00…ask_sz_09` (corresponding sizes)  
  - Optional timestamps (`ts_event`, `ts_recv`)  

**Preprocessing steps:**

- Unzip and load each CSV into pandas.  
- Drop any rows missing best bid/ask.  
- Compute the mid-price at each snapshot as  
  m_t = (bid_t,0 + ask_t,0) / 2

---

## 2. Temporary Impact Definition

When a market order of size \(X\) shares arrives at time \(t\), it “walks” through the book levels:

- **Buy orders** consume ascending ask levels \((p_i, q_i)\).  
- **Sell orders** consume descending bid levels analogously.

The **temporary impact** measures the average per-share price concession:

g_t(X) = (1 / X) * sum_{i=1..k} [ (p_i - m_t) * q_i ]

where \(m_t\) is the mid-price.  This definition captures exactly how slippage grows as larger blocks interact with thinner liquidity at deeper levels.

**Illustrative toy example:**  
| Level | Ask Price \(p_i\) | Ask Size \(q_i\) |
|:-----:|:-----------------:|:----------------:|
| 1     | \$10.00           | 50 shares        |
| 2     | \$10.05           | 75 shares        |
| 3     | \$10.10           | 100 shares       |

The bid side might mirror \$9.95/75, \$9.90/50, …, giving a mid-price of \$9.975. A 100-share buy would clear 50 at \$10.00 and 50 at \$10.05, yielding an average slippage of \$0.05 per share.

---

## 3. Q1 – Empirical Modeling of \(g_t(X)\)

### Goals

- **Describe** how slippage grows with order size \(X\).  
- **Ensure** models are both **expressive** and **tractable** for optimization.

### Candidate Models

1. **Linear model:**  
   g_t(X) ≈ beta_t * X
   - **When to use:** If book depth is relatively uniform.  
   - **Why:** Simplest form; yields a linear cost function amenable to fast convex solvers.  
   - **Limitation:** Understates impact for large \(X\) that traverse deeper, thinner levels.

2. **Power-law model:**  
   g_t(X) = alpha_t * X^beta_t    (0 < beta_t < 1)
   - **When to use:** For larger trades where slippage accelerates.  
   - **Why:** Captures concave growth—initial shares clear thick levels cheaply, later shares pay more.  
   - **Limitation:** Requires nonlinear fitting and solver support for convex power functions.

### Fitting Procedure

- Evaluate (X, g_t(X)) on a grid X = 100, 200, …, 1000.  
- **Linear fit:** Ordinary least squares through the origin → beta_t.  
- **Power-law fit:** Nonlinear least squares → (alpha_t, beta_t).  
- Compare goodness-of-fit R^2 for both models.

### Summary of Results

| Symbol | Median β (linear) | Median (α,β) power-law | ΔR² (lin→pl) |
|:------:|:-----------------:|:----------------------:|:------------:|
| SOUN   | 4.5 × 10⁻⁵        | (2.5 × 10⁻⁴, 0.74)     | +3%          |
| CRWV   | 9.0 × 10⁻⁴        | (0.36, 0.09)           | +1%          |
| FROG   | 2.4 × 10⁻³        | (0.46, 0.20)           | +4%          |

- **Interpretation:**  
  - SOUN’s low β indicates deep liquidity (low per-share cost).  
  - FROG’s high β indicates shallow book (high slippage).  
- **Model choice:**  
  - **Linear** for speed‐sensitive, large‐scale optimization.  
  - **Power-law** for high-accuracy cost forecasting on bulk trades.

---

## 4. Q2 – Execution-Optimization Framework

### Mathematical Formulation

Given total target \(S\) and \(N\) decision times t_1,…,t_N:, allocate \(\{x_i\}\) to minimize total impact:

minimize   sum_{i=1..N} g_{t_i}(x_i)
subject to sum_{i=1..N} x_i = S,   x_i >= 0

### Linear-Cost Special Case

- **Cost:** g_{t_i}(x) = beta_i * x.  
- **Solution:** Allocate all \(S\) to the period with minimum beta_i.  
- **Why this matters:** Illustrates how uniform execution can be suboptimal if liquidity varies.

### Power-Law Generalization

- **Cost:** g_{t_i}(x) = alpha_i * x^{beta_i} (convex for beta_i).  
- **Lagrangian:**  
  L(x, λ) = sum_i alpha_i * x_i^{beta_i}  - λ*(sum_i x_i - S).  
- **First-order condition:**  
  alpha_i * beta_i * x_i^{beta_i - 1} = λ   for all i  
- **Interpretation:**  
  Water-filling: allocate more to periods with lower marginal cost.

### Practical Extensions

- **Pacing constraints:** Cap \(x_i\le U\) or add quadratic penalties to enforce smooth execution.  
- **Dynamic programming:** Model stochastic \(g_t\) over time for adaptive strategies.  
- **Risk constraints:** Incorporate variance or Value-at-Risk into a mean–variance trade-off.

---


