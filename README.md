# Fraud Analysis in Card-Not-Present Transactions

![Python](https://img.shields.io/badge/Python-pandas%20%7C%20seaborn-blue)
![SQL](https://img.shields.io/badge/SQL-DuckDB-orange)
![LLM](https://img.shields.io/badge/LLM-review_agent-purple)
[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/mathfnz/case-cloudwalk-antifraude/blob/main/cloudwalk-fraud-analysis.ipynb)

Analysis of 3,199 card-not-present transactions to identify suspicious behaviour, recommend preventive measures and design an anti-fraud solution. Technical case for the Martech Analyst process at CloudWalk.

## Headline numbers

| | |
|---|---|
| 🔥 **12.2%** | portfolio chargeback rate — **12x** the ~1% network threshold |
| 💸 **BRL 568k** | disputed: 23% of all money processed |
| 🎯 **86 accounts** | 3% of the base concentrate **83%** of every chargeback |
| 🃏 **31 cards** | on a single account, across 4 devices — 81% chargeback |
| ✅ **93%** | of the portfolio is clean and should feel zero friction |

## The problem

The sample covers one month (Nov 2019) and BRL 2.46M processed. The first measurement sizes the problem: **12.2% of transactions ended in a chargeback**, against the ~1% threshold where card networks start monitoring programmes and fines. In monetary terms it is worse — **BRL 568k disputed, 23% of all money processed**.

Because every transaction is card-not-present, fraud liability falls on the merchant side, and on the acquirer as guarantor when the merchant cannot cover it. Fraud here is a direct hit to cash.

## Method

Every finding follows **hypothesis → evidence → action**. Seven hypotheses were stated before testing, each derived from a known mechanism of CNP fraud and each predicting a measurable trace in the data. Six were confirmed or refined; one was refuted and is documented as such.

Tools: **SQL (DuckDB over pandas)** for all exploration, seaborn for visualisation, and an LLM layer for the review-agent prototype.

## The investigation, in four turns

**I started where the data looked broken.** `device_id` was missing in 26% of rows — first read as a quality problem, then as possible evasion. **Wrong**: the group without a device shows a *lower* chargeback rate (8.1% vs 13.7%). Not camouflage — a collection gap.

**So I inverted the question.** If he does not hide in missing data, he is visible in some excess. Grouping by device surfaced one handset with **22 distinct cards**. It could have been a store — but 19 of those 22 transactions were disputed. No legitimate store accumulates 86%.

**A column I was not looking at changed the direction.** Every suspicious device belonged to a single account. The device is disposable; the login is not. Re-running by account surfaced the largest operator whole: **31 distinct cards, 4 devices, 81% chargeback** — in the device view he had dissolved into four small piles.

**Then I followed his trail** to the other side of the counter: 23 of his 31 purchases went to one merchant, and that merchant shows **100% chargeback with three buyers**. Not a fraudster using a store — an operation on both sides, with the store as the cash-out.

## The profile that emerged

> Not the diffuse opportunist. The professional operator.

- 🪪 **Dedicated accounts** — keeps the login, changes devices (the account is the stable signature)
- 🃏 **Lists of stolen cards** — up to 31 distinct cards on one account; 4+ uses of the same card reach 55.6% chargeback, reuse within an hour 60.3%
- 💰 **Buys liquidity** — above BRL 2,000 the rate triples against baseline (33.6%)
- 🌙 **Prefers late night** — 17.3% between midnight and 6am against 4.5% in the morning
- 🔁 **Acts in series** — first purchase 4.8%, second onwards 52.5%
- 🏪 **At the top, collusion** — shell merchants with 100% chargeback and two or three buyers

## What I propose

**Risk tiers per account.** 93% of the base is clean and passes with zero friction; **86 accounts (3%) hold 83% of the problem** and go to immediate block; a watchlist tier faces a mandatory 3DS challenge. Materialised as a queryable table, scheduled in production.

**Anti-fraud architecture in three layers.** Deterministic rules settle the obvious in milliseconds; a risk score handles the grey zone and routes it to a challenge; a feedback loop retrains as chargebacks confirm.

**Review agent with a human in the loop.** A prototype that assembles the transaction dossier in SQL — deterministic and auditable, the language model never calculates or invents a number — and uses an LLM only to draft the analyst's opinion. The decision stays human: the model cannot see context outside the data, the cost of erring is asymmetric, and accountability cannot be delegated to a model.

The principle that calibrates everything: **a false negative costs the chargeback; a false positive costs the sale, the revenue and the customer.** Which makes fraud control a retention problem too — every legitimate customer wrongly blocked is churn.

## The data that would make the biggest difference

A **hashed cardholder identity** on both account and card would answer the decisive question directly — *is the cardholder the account owner?* — characterising third-party use on the first transaction, without waiting weeks for a chargeback. **Location and IP type** would separate one person with four devices from a ring sharing a login. And **account age**, which says little alone, becomes powerful crossed with the chargeback analysis: it distinguishes loyal recurrence from the compressed recurrence of a three-day-old account.

## How to run

1. Open the notebook in [Google Colab](https://colab.research.google.com) — or use the badge above;
2. Upload `transactional-sample.csv` to the file sidebar;
3. Runtime → Run all.

The review-agent cell requires your own Groq API key, stored as a Colab secret named `GROQ_API_KEY`. No credentials are versioned in this repository. Results are saved in the notebook — running it is only needed to reproduce or modify the analysis.

## Repository

| File | Description |
|---|---|
| `cloudwalk-fraud-analysis.ipynb` | Full analysis: context, data profiling, hypotheses, exploratory analysis, insights, executive report and appendix |
| `transactional-sample.csv` | Transaction sample provided with the case |
| `classified_transactions.csv` | Output: every transaction labelled with its account's risk tier |

## Declared limitations

One month of data — no seasonality, and the recurrence rule needs calibration against a longer history. The chargeback label matures late (disputes arrive weeks later), so 12.2% is a floor. The card mask can collapse distinct cards, meaning the method undercounts and never overcounts. And the sample holds approved transactions only — declines, the live trace of card testing, are absent.
