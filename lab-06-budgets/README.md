# AWS Cloud Bootcamp — Lab 6: Account Budgets & Cost Alerts

## What This Is

This lab applies AWS cost-management concepts by configuring monthly budgets, tiered cost alerts, a zero-spend alert, SNS-based notifications, and exploring AWS Cost Explorer.

  
**Interface:** AWS Console  
**Goal:** Implement cost monitoring and proactive alerting to improve visibility and control of AWS spending.

---

## What I Built

| Component | Setting | Purpose |
|-----------|---------|---------|
| `monthly-cost` budget | $20/month, recurring | Tracks total monthly spend |
| 50% actual alert | Fires at $10 spent | Early heads-up |
| 80% actual alert | Fires at $16 spent | Close to limit — investigate |
| 100% forecasted alert | Fires when on track to hit $20 | Warning before overspend |
| Zero-spend budget | $0.01 threshold | Tripwire for any charge at all |
| Cost Explorer | Grouped by service, daily granularity | Visibility into where money goes |

---

## Key Things I Learned

**Budgets alert on spend — the Free Tier just reduces what you're charged.**  
They serve different purposes. You need both.

**Forecasted alerts are the more useful ones.**  
They fire before you overspend based on your run-rate, not after.

**A zero-spend budget is the best safety net for a sandbox account.**  
Any unexpected charge triggers it within hours.

**SNS decouples the alert from the destination.**  
Route alerts to an SNS topic once, then fan out to email, Slack, or Lambda
without touching the budget config.

**Cost Explorer needs activated cost allocation tags to be meaningful.**  
Tag resources at creation — retroactive tagging doesn't backfill historical data.

---

## Key Concepts

**1. Actual vs. forecasted threshold** — actual fires after money is spent;
forecasted fires when you're *on track* to overspend. Use both.

**2. Zero-spend budget** — pre-configured template with a $0.01 threshold.
Ideal for training or personal accounts expected to stay free.

**3. SNS fan-out pattern** — one SNS topic, many subscribers. Keeps alert
destinations flexible without changing the budget itself.

**4. Cost allocation tags** — must be activated in the Billing console before
Cost Explorer can group or filter by them.

**5. Cost Explorer granularity** — monthly view answers "how much this month";
daily view answers "which day did usage spike".

---

## Files in This Lab

| File | What It Is |
|------|-----------|
| `README.md` | This file — summary and reflections |
| `account-budgets-cost-alerts-lab.md` | Full concept notes and step-by-step lab walkthrough |

---

## Lab Status

- [x] Monthly cost budget created with 50% / 80% actual and 100% forecasted alerts
- [x] Zero-spend budget created as a tripwire
- [x] Alert emails confirmed (inbox checked, subscription confirmed)
- [x] Cost Explorer launched and grouped by service
- [x] Difference between actual and forecasted alerts understood

## Screenshots

- [x] Billing console — Budgets dashboard showing both budgets (monthly-cost and zero-spend) → `screenshots/01-budgets-dashboard.png`
- [x] Billing console — monthly-cost budget detail showing all three alert thresholds → `screenshots/02-alerts-threshold.png`
- [x] Cost Explorer — spend grouped by service → `screenshots/03-cost-explorer-grouped-by-service.png`

---

*Part of my AWS Cloud Bootcamp — see the root [README](../README.md) for all labs.*
