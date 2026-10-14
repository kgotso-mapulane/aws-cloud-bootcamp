# AWS Cloud Bootcamp — Lab 6: Account Budgets & Cost Alerts

**Duration:** 30–40 min  
**Level:** Some IT background  
**Interface:** AWS Console  
**Cost:** AWS Free Tier

---

## Why This Matters

After Lab 5 you have an ALB and an Auto Scaling Group running — resources that
bill by the hour. Budgets are how you find out about spend *before* the monthly
invoice does. This lab makes cost visible and puts alerts in place so nothing
surprises you.

> **Budgets vs. the Free Tier**  
> The Free Tier and credits limit what you're charged, but they don't alert you
> as usage climbs. A budget watches actual and forecast spend and emails you at
> thresholds you set — the early-warning system credits alone don't give you.

---

## Learning Objectives

- Create a monthly cost budget with multiple alert thresholds (actual and forecasted).
- Create a zero-spend budget that fires the moment any real charge appears.
- Route alerts to email and understand how to fan out to Slack or webhooks.
- Use Cost Explorer to see where money is going.

---

## Prerequisites

- Day One Lab 1 complete (IAM admin user).
- Access to Billing and Cost Management. If you get an access-denied error as
  an IAM user, the root user must enable IAM access to Billing under
  **Account → Account settings**.

---

## Part A — Monthly Cost Budget

### Step 1 · Create the Budget

1. Open **Billing and Cost Management → Budgets → Create budget**.
2. Choose **Customize (advanced) → Cost budget**. Click **Next**.
3. Name it `monthly-cost`. Set Period to **Monthly**, Recurring.
4. Set a budgeted amount — e.g. **$20** (adjust to fit your class).

### Step 2 · Add Alert Thresholds

Add several thresholds on the alerts step so you hear about trouble early:

| Threshold | Type | Fires when… |
|-----------|------|-------------|
| 50% of budget | Actual | You've spent half — a heads-up |
| 80% of budget | Actual | Getting close — investigate now |
| 100% of budget | Forecasted | AWS predicts you'll blow the budget this month |

5. For each threshold click **Add alert threshold**, set the percentage and
   the **Trigger** (Actual or Forecasted), and add your email as the recipient.
6. **Review and Create budget**.

> **Forecasted alerts are the useful ones**  
> A forecasted alert warns you *before* you overspend, based on your run-rate.
> An actual alert only fires after the money is gone. Use both, but pay
> attention to the forecast.

---

## Part B — Zero-Spend Budget

A zero-spend budget alerts you the instant AWS bills you anything at all — the
best tripwire for a training or sandbox account that's supposed to stay free.

7. **Budgets → Create budget → Use a template (simplified) → Zero spend budget**.
8. Add your email and **Create budget**. It sets a $0.01 threshold automatically.

> **Layer your budgets**  
> Run both: the zero-spend budget catches any charge at all, while the $20
> monthly budget with forecasting tells you when normal usage is trending high.
> They serve different purposes.

---

## Part C — Richer Alerting & Visibility

### Step 3 · Send Alerts to a Team Channel (Concept)

Email is fine for individuals; teams usually want alerts in a channel. In the
budget's alert step you can add an **Amazon SNS topic** as a recipient instead
of (or alongside) email. Once alerts land in SNS you can fan them out to Slack
or Teams via a webhook, to SMS, or to a Lambda — without changing the budget.

To set this up later:
- **SNS console → Create topic** (Standard)
- Add subscriptions (email / HTTPS webhook)
- Reference that topic's ARN in the budget alert

No CLI required — it's all in the SNS and Budgets consoles.

### Step 4 · Explore Spend in Cost Explorer

9. **Billing → Cost Explorer → Launch Cost Explorer** (first launch takes a
   few minutes to populate).
10. **Group by Service** to see what costs the most.
11. Switch granularity to **Daily** to spot a spike on the day it happened.
12. **Filter by Tag** (e.g. `Project=bootcamp`) to attribute cost to this course.

> **Tag as you go**  
> Cost allocation is only as good as your tags. Tagging resources with
> `Project` / `Owner` / `Environment` now makes Cost Explorer reports
> meaningful later. Activate cost allocation tags under
> **Billing → Cost allocation tags**.

---

## Cleanup & Teardown

Budgets, Cost Explorer, and SNS topics are effectively free (the first two
budgets per month are free; extra budgets are ~$0.02/day). You can safely keep
the budgets — they're the whole point.

| Resource | Keep or Delete | Where in the Console |
|----------|---------------|----------------------|
| `monthly-cost` budget | Keep (recommended) | Billing → Budgets → select → Delete |
| Zero-spend budget | Keep (recommended) | Billing → Budgets → select → Delete |
| Any demo SNS topic | Delete if unused | SNS → Topics → select → Delete |

> **Don't forget Lab 5**  
> Budgets watch spend — they don't stop it. If you haven't already, go back
> and tear down the ALB and Auto Scaling Group from Lab 5; those are the
> resources that actually cost money.

---

## Key Things I Learned

**Budgets are the early-warning system that credits alone don't provide.**  
Free Tier credits mean you might not be charged, but you can still accumulate
usage that would cost money without them. Budgets alert you to that usage while
you can still act on it.

**Forecasted alerts fire before you overspend — actual alerts fire after.**  
The forecasted alert uses your current run-rate to project the full month. It's
the more useful of the two for prevention. Actual alerts confirm what already
happened.

**Zero-spend budgets are the best tripwire for sandbox accounts.**  
If a training account is supposed to cost nothing, any charge at all is
unexpected. The $0.01 threshold means you hear about it within hours, not at
end of month.

**SNS decouples the alert from the destination.**  
By routing alerts through SNS, you can change where they go (email, Slack,
Lambda) without touching the budget config. This is a reusable pattern across
many AWS services — CloudWatch alarms, EventBridge rules, and others all
support SNS as a target.

**Cost Explorer is only as useful as your tags.**  
Without consistent tags, every chart just shows undifferentiated spend. Tagging
at resource creation — not retroactively — is the habit that makes cost
visibility actually work.

---

## What Confused Me Initially

- Why the Free Tier doesn't send alerts on its own — the Free Tier is a
  billing discount, not a monitoring tool. It prevents charges; it doesn't
  observe usage. Those are two different jobs.
- Why forecasted alerts sometimes don't fire early in the month — AWS needs
  enough daily data points to build a meaningful forecast. In the first few
  days of a month the projection can be unreliable; it improves as the month
  progresses.

---

## Key Concepts

**1. Actual vs. forecasted threshold** — actual fires after spend reaches the
percentage; forecasted fires when the projected end-of-month spend is on track
to exceed it. Use both for full coverage.

**2. Zero-spend budget** — a pre-configured template that alerts on any charge
at all. Ideal for sandbox, training, or personal accounts expected to stay free.

**3. SNS fan-out pattern** — one SNS topic can have many subscriptions (email,
HTTPS, SMS, Lambda). Routing budget alerts to SNS instead of directly to email
keeps the destination flexible and team-friendly.

**4. Cost allocation tags** — tags you activate in the Billing console that
Cost Explorer can filter and group by. Without activating them, Cost Explorer
cannot use them even if the tags exist on resources.

**5. Cost Explorer granularity** — switching between monthly and daily view
is how you move from "how much did I spend this month" to "which day did usage
spike and why".

---

## Files in This Lab

| File | What It Is |
|------|-----------|
| `README.md` | Summary, status checklist, and reflections |
| `account-budgets-cost-alerts-lab.md` | This file — full concept notes and step-by-step walkthrough |

---

*Part of my AWS Cloud Bootcamp — see the root [README](../README.md) for all labs.*
