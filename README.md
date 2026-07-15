# RefillRx

**Retention isn't monthly. It's per milligram.**

Live demo → [samson1402.github.io/refillrx](https://samson1402.github.io/refillrx)

---

## The problem, in plain words

Almost every subscription business measures retention by **calendar month**. Month 1, month 2, month 3. How many people are still here?

For most businesses that's fine. For a GLP-1 weight-loss subscription, it hides the only thing that matters.

Here's why. A GLP-1 treatment doesn't start at full strength. It climbs:

```
0.25 mg  →  0.5 mg  →  1.0 mg  →  1.7 mg  →  2.4 mg
 4 weeks    4 weeks    4 weeks    4 weeks    maintenance
```

Each step up means more side effects — nausea, stomach trouble — before it means more results. So people don't quit on a random Tuesday in month 3. **They quit at a dose.**

Two customers quit at 1.0 mg. One did it in week 9. The other did it in week 14. Month-based retention puts them in different buckets and tells you nothing. Dose-based retention puts them in the same bucket and tells you exactly where your product hurts.

The question "why did month 3 drop?" has no answer. The question "why do 11% of people quit at 0.5 mg?" has an answer, and someone can act on it.

---

## What RefillRx does

It re-cuts retention by **titration step** instead of calendar month, and shows four things:

### 1. The titration ladder

The main chart. A staircase where:

- each flat step = a dose
- the height of the step = how much of the cohort is still there
- the **red vertical drop** = the customers you lost moving to the next dose

You look at it once and you know which milligram is the problem. In the demo, it's usually the jump from 0.25 mg to 0.5 mg — the first real increase, and the first time side effects show up.

### 2. Pause is not churn

Wellis lets people pause for free. That's a good product decision and a terrible reporting problem, because most dashboards count a pause as a loss.

It isn't. About a third of paused customers come back within 90 days. If you count them as churned, you'll spend money re-acquiring people who were coming back anyway.

RefillRx splits every exit into **active / paused / cancelled** and tracks how many of the paused ones return.

### 3. Revenue retention vs customer retention

These two numbers move in opposite directions, and that's the whole business model.

- **Customer retention falls.** People leave. That's normal.
- **Revenue retention rises** — because the people who stay climb the dose ladder, and a higher dose costs more.

The gap between the two lines is dose escalation paying for churn. If revenue retention is above 100%, the business grows even while losing customers. If it drops below, it doesn't. That single number is arguably the most important one at the company, and you can't see it at all with month-based retention.

### 4. Cohort heatmap

The standard view, kept for the people who want it. Start month down the side, months on treatment across the top. Useful for spotting whether newer cohorts behave differently — for example, whether the German cohorts fall off faster than the Dutch ones.

---

## The one number I'd put on a wall

**Median doses to churn.**

Not "months to churn". Pens. How many pens does the average person who leaves take before they leave?

If it's 4, your problem is side effects at the low doses and it's a medical guidance problem. If it's 9, your problem is that people hit their goal weight and stop, and it's a maintenance-product problem.

Same churn rate. Completely different companies. Month-based retention can't tell them apart.

---

## Decisions worth arguing about

In the code as comments, because they're judgement calls:

**42 days = a lapse.**
A pen lasts 28 days. Add a 14-day grace window and you get 42. Past that, the customer has lapsed. This number came from Medical Ops. If they disagree, one line changes and every chart follows.

**Trust the shipment, not the flag.**
Firmhouse will happily leave a subscription marked `active` after the pharmacy has stopped dispensing to that person. The flag lies. The dispense record doesn't. Every model in this repo derives state from what actually left the warehouse.

**Approval rate is not a funnel metric.**
The percentage of intakes a doctor approves is a **medical decision**. It appears in the demo because it's useful context. It should never be optimised, and it should never appear in a growth target. That's not a technical position — it's the difference between a health company and a webshop.

---

## How it's built

- **BigQuery** — the warehouse
- **dbt** — 4 models, 11 tests
- **Python** — generates synthetic subscription and dispense data so the repo runs standalone
- **Looker** — where this lives in real life

The key model is `int_titration_steps.sql`. It uses window functions to turn a flat list of dispenses into a dose journey per customer:

```sql
dense_rank() over (
    partition by subscription_id
    order by dose_mg
) as titration_step
```

That one line is the whole idea. Everything else is presentation.

---

## Repo layout

```
refillrx/
├── models/
│   ├── staging/
│   │   ├── stg_firmhouse__subscriptions.sql   # state, with the 'active' flag distrusted
│   │   ├── stg_pharmacy__dispenses.sql        # what actually shipped
│   │   └── stg_shopify__order_lines.sql       # the money
│   ├── intermediate/
│   │   └── int_titration_steps.sql            # the dose journey — the core idea
│   └── marts/
│       ├── fct_refill_retention.sql           # one grain, three charts
│       └── schema.yml                         # 11 tests
├── analysis/
│   └── median_doses_to_churn.sql
├── generate_data.py
├── demo/index.html
└── README.md
```

One grain, three charts: the ladder, the heatmap and the NRR line all read from `fct_refill_retention`. One definition means Finance and Marketing stop arguing about what churn means.

---

## Run it

```bash
git clone https://github.com/SamSon1402/refillrx.git
cd refillrx

pip install -r requirements.txt
python generate_data.py        # ~6,000 synthetic subscriptions across 3 treatments, 2 markets

dbt run
dbt test
```

Or just open `demo/index.html` in a browser. Nothing to install.

---

## A note on the data

**Every number here is invented.** No Wellis data was used, seen, or inferred.

The dose ladders are real — those are the published titration schedules for Wegovy, Ozempic and Mounjaro, which anyone can look up. Everything else is synthetic: the retention rates, the cohort sizes, the euro amounts.

The point of this repo is **which question to ask**, not the answer.

---

Built by [Sameer Mohammad](https://github.com/SamSon1402) — Paris.
