# Fraud Analysis in Card-Not-Present Transactions

![Python](https://img.shields.io/badge/Python-pandas%20%7C%20seaborn-blue)
![SQL](https://img.shields.io/badge/SQL-DuckDB-orange)
![LLM](https://img.shields.io/badge/LLM-review_agent-purple)
[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/mathfnz/case-cloudwalk-anti-fraud/blob/main/cloudwalk-fraud-analysis.ipynb)

Analysis of 3,199 card-not-present transactions to identify suspicious behaviour, recommend preventive measures and design an anti-fraud solution. Technical case for the Martech Analyst process at CloudWalk.

**This README summarises the notebook section by section.** Full analysis in [`cloudwalk-fraud-analysis.ipynb`](cloudwalk-fraud-analysis.ipynb).

| | |
|---|---|
| 🔥 **12.2%** | portfolio chargeback rate — **12x** the ~1% network threshold |
| 💸 **BRL 568k** | disputed: 23% of all money processed |
| 🎯 **86 accounts** | 3% of the base concentrate **83%** of every chargeback |
| 🃏 **31 cards** | on a single account, across 4 devices — 81% chargeback |
| ✅ **93%** | of the portfolio is clean and should feel zero friction |

## 1. Context and objective

The sample covers one month (Nov 2019) and BRL 2.46M processed. The first measurement sizes the problem: **12.2% of transactions ended in a chargeback**, against the ~1% threshold where card networks start monitoring programmes and fines. In monetary terms it is worse — **BRL 568k disputed, 23% of all money processed**.

Because every transaction is card-not-present, fraud liability falls on the merchant side, and on the acquirer as guarantor when the merchant cannot cover it. Fraud here is a direct hit to cash.

**Method:** every finding follows **hypothesis → evidence → action**. Tools: SQL (DuckDB over pandas) for exploration, seaborn for visualisation, an LLM layer for the review-agent prototype. AI was used as an analytical partner; the hypotheses and conclusions are mine.

## 2. About the data

Eight columns, 3,199 rows, one month. Two things worth flagging up front: `device_id` is **the buyer's device** (all transactions are card-not-present, so no terminal is involved), and `has_cbk` is **the label** every hypothesis is judged against.

The notebook includes an automated quality battery — uniqueness, nulls in critical fields, positive amounts, date continuity, card-mask format and `device_id` coverage. One legitimate alert fired: coverage at 74%, far below threshold. One alert fired on a wrong premise (14 to 17-digit cards flagged as malformed) and the rule was refined — stated expectation → alert → investigation → better rule.

## 3. Hypotheses

Seven, stated before testing, each derived from a known mechanism of CNP fraud and each predicting a measurable trace: fraudsters hiding via missing device data (H1), card testing from a single point (H2), concentration in few operators (H3), the same card hammered repeatedly (H4), high-value purchases (H5), late-night activity (H6) and merchant collusion (H7).

Six were confirmed or refined. **One was refuted and stayed in the notebook** — method is also a result.

## 4. Exploratory analysis — the investigation in four turns

**I started where the data looked broken.** `device_id` missing in 26% of rows — first read as a quality problem, then as possible evasion (H1). **Wrong**: that group shows a *lower* chargeback rate (8.1% vs 13.7%). Not camouflage — a collection gap, which still blinds the engine on one in four transactions.

**So I inverted the question.** If he does not hide in missing data, he shows up in some excess. Grouping by device surfaced one handset with **22 distinct cards** — it could have been a store, but 19 of those 22 transactions were disputed. No legitimate store accumulates 86%.

**A column I was not looking at changed the direction.** Every suspicious device belonged to a single account. The device is disposable; the login is not. Re-running by account surfaced the largest operator whole: **31 distinct cards, 4 devices, 81% chargeback** — in the device view he had dissolved into four small piles.

**Then I followed his trail** to the other side of the counter: 23 of his 31 purchases went to one merchant, and that merchant shows **100% chargeback with three buyers**. Not a fraudster using a store — an operation on both sides, with the store as the cash-out.

Supporting findings from the same section: amount (above BRL 2,000 the rate reaches **33.6%**, against 3-4% up to BRL 500), hour (**17.3%** late night against 4.5% in the morning), recurrence (**4.8% → 52.5%** from the second purchase in the month) and the hammered card (**7.3% → 55.6%** from one use to four or more; **60.3%** when reused within an hour).

## 5. Insights

> Not the diffuse opportunist. The professional operator.

- 🪪 **Dedicated accounts** — keeps the login, changes devices (the account is the stable signature)
- 🃏 **Lists of stolen cards** — up to 31 on one account, hammered until exhausted
- 💰 **Buys liquidity** — above BRL 2,000 the rate triples against baseline
- 🌙 **Prefers late night** — 17.3% against 4.5% in the morning
- 🔁 **Acts in series** — first purchase 4.8%, second onwards 52.5%
- 🏪 **At the top, collusion** — shell merchants with 100% chargeback and two or three buyers

The profile becomes policy through **risk tiers per account**: **A (93%)** clean, zero friction; **B (3%)** convicted by the data — 8+ distinct cards, 3+ chargebacks or a rate above 50% — holding **83% of all chargebacks**, blocked immediately; **C** the ambiguous shape, under mandatory 3DS challenge. Tier A came out at exactly zero chargebacks, meaning the criteria captured all 391 disputes. The classification is materialised as a queryable table (`classified_transactions.csv`).

## 6. Executive report

**Immediate:** block the 86 tier-B accounts and hold settlement for merchants matching the collusion pattern — retaining pending payouts closes the cash-out window the scheme depends on.

**Prevention rules:** graduated ladder on distinct cards per account (3-4 monitor, 5-7 challenge with 3DS, 8+ decline); challenge above ~BRL 1,000 combined with any second signal; card velocity; progressive recurrence scoring.

**Structural:** fix `device_id` instrumentation and monitor chargeback per merchant with automatic settlement holds.

**The solution** is three layers — deterministic rules for the obvious in milliseconds, a risk score for the grey zone routed to a challenge, and a feedback loop that retrains as chargebacks confirm — plus a **review agent with a human in the loop**: SQL builds the dossier (deterministic and auditable, the model never calculates or invents a number), an LLM drafts the opinion, the analyst decides. The model cannot see context outside the data, the cost of erring is asymmetric, and accountability cannot be delegated to a model.

The principle that calibrates everything: **a false negative costs the chargeback; a false positive costs the sale, the revenue and the customer** — which makes fraud control a retention problem too.

## 7. Appendix

**The payments industry** — money and information flows, the five players, acquirer vs sub-acquirer vs gateway, chargebacks against cancellations, and what an anti-fraud system is (answers to the case's theory section).

**Data that would strengthen the analysis** — led by a **hashed cardholder identity** on account and card, which answers directly whether the cardholder is the account owner, characterising third-party use on the first transaction instead of waiting weeks for the dispute; **location and IP type**, which would settle whether those 4 devices were one person or a ring; and **account age**, which says little alone but crossed with the chargeback analysis separates loyal recurrence from a three-day-old account operating in series. Turned into a concrete event tracking spec with naming conventions and required properties.

**Declared limitations** — one month of data (no seasonality; the recurrence rule needs a longer history), a label that matures late (12.2% is a floor), a card mask that undercounts distinct cards, and approved transactions only, so declines are absent.

## How to run

1. Open the notebook in [Google Colab](https://colab.research.google.com) — or use the badge above;
2. Upload `transactional-sample.csv` to the file sidebar;
3. Runtime → Run all.

The review-agent cell requires your own Groq API key, stored as a Colab secret named `GROQ_API_KEY`. No credentials are versioned in this repository. Results are saved in the notebook — running it is only needed to reproduce or modify the analysis.

## Repository

| File | Description |
|---|---|
| `cloudwalk-fraud-analysis.ipynb` | Full analysis, structured as sections 1 to 7 above |
| `transactional-sample.csv` | Transaction sample provided with the case |
| `classified_transactions.csv` | Output: every transaction labelled with its account's risk tier |
