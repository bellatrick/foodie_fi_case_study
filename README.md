# CASE STUDY: FOODIE-FI
![image](https://github.com/bellatrick/foodie_fi_case_study/assets/74540938/776332fa-e3d5-466b-b32b-8dafff339893)

## Introduction

Foodie-Fi is a subscription-based streaming service specializing in food-related content, similar to Netflix but focused exclusively on cooking shows. In 2020, Danny and his team launched this startup, offering monthly and annual subscriptions, giving customers unlimited access to exclusive food videos worldwide. Danny aimed to make data-driven decisions for investment and feature development. This case study delves into using digital subscription data to answer key business questions.

There are two tables available for analysis, namely:

![image](https://github.com/bellatrick/foodie_fi_case_study/assets/74540938/767fde02-d883-47dd-959d-f98611ceea72)

## Table 1: `plans`
![image](https://github.com/bellatrick/foodie_fi_case_study/assets/74540938/164578a6-ba09-4a43-938e-740cff927e63)

Customers select from various plans upon signing up for Foodie-Fi.

- Basic plan offers limited access and costs $9.90 per month.
- Pro plan has no watch time limits, supports offline video downloads, and costs $19.90 per month or $199 annually.
- Customers start with a 7-day free trial and transition to the pro monthly plan unless they cancel, switch to the basic plan, or upgrade to the annual pro plan during the trial.
- When customers cancel, they receive a churn plan record with a null price, and their plan remains active until the billing period ends.

## Table 2: `subscriptions`
![image](https://github.com/bellatrick/foodie_fi_case_study/assets/74540938/9b05901e-ec86-4955-b1a8-c73b2c21de71)

Customer subscriptions include the start date of their specific plan_id.

- If customers downgrade or cancel their subscription, the higher plan continues until the period concludes.
- When customers upgrade from a basic to a pro or annual pro plan, the new plan becomes effective immediately.
- Churning customers retain access until the billing period ends, but the start_date reflects the cancellation date.

## Case Study Questions

From the two tables available, Danny needs answers to enable him to expand his business to a larger audience next year. You can review the questions and answers below:

### A. Customer Journey

Based off the 8 sample customers provided in the sample from the subscriptions table, write a brief description about each customer’s onboarding journey.

I will keep the customer journey restricted to the first 3 customers:

- **Customer ID 1**: Started the free trial on 2020-08-01. Subscribed to the basic monthly plan after the trial.
- **Customer ID 2**: Began the trial on 2020-09-20 and subsequently subscribed to the pro annual plan.
- **Customer ID 8**: Initiated the trial on 2020-06-11. After the trial, they subscribed to the basic monthly plan. Two weeks later, they subscribed to the pro monthly plan on 2020-08-03.

### B. Data Analysis Questions

1. **How many customers has Foodie-Fi ever had?**

   - SQL Code:
     ```sql
     SELECT COUNT(DISTINCT customer_id) as "total customers"
     FROM f_view;
     ```
   - Answer: 1000 customers

2. **What is the monthly distribution of trial plan start_date values for our dataset?**

- SQL Code:

```sql
SELECT
date_format(start_date,"%y-%m-01") AS start_month,
monthname(start_date) AS month_name,
COUNT(*) AS trial_counts
FROM f_view
WHERE plan_name='trial'
GROUP BY date_format(start_date,"%y-%m-01"),month_name
ORDER BY trial_counts DESC;
```

| start_month | month_name | trial_counts |
| ----------- | ---------- | ------------ |
| 20-01-01    | January    | 88           |
| 20-02-01    | February   | 68           |
| 20-03-01    | March      | 94           |
| 20-04-01    | April      | 81           |
| 20-05-01    | May        | 88           |
| 20-06-01    | June       | 79           |
| 20-07-01    | July       | 89           |
| 20-08-01    | August     | 88           |
| 20-09-01    | September  | 87           |
| 20-10-01    | October    | 79           |
| 20-11-01    | November   | 75           |
| 20-12-01    | December   | 84           |

- Highest in March (94 trials) and lowest in February (68 trials).

3. **What plan start_date values occur after the year 2020?**

   - SQL Code:
     ```sql
     SELECT
     plan_name,
     COUNT(*) AS total_plans
     FROM f_view
     WHERE start_date > "2020-12-31"
     GROUP BY plan_name
     ORDER BY total_plans;
     ```
   - 8 basic monthly, 60 pro monthly, 63 pro annual, and 71 churn plans.

   | plan_name     | total_plans |
   | ------------- | ----------- |
   | Basic monthly | 8           |
   | Pro monthly   | 60          |
   | Pro annual    | 63          |
   | Churn         | 71          |

4. **What is the customer count and percentage of customers who have churned?**

   - SQL Code:
     ```sql
     SELECT
     COUNT(DISTINCT CASE
             WHEN plan_name = 'churn' THEN customer_id
         END) AS total_customers,
     ROUND(COUNT(DISTINCT CASE
                     WHEN plan_name = 'churn' THEN customer_id
                 END) / COUNT(DISTINCT customer_id) * 100,
             1) AS churn_percentage
     FROM f_view;
     ```
   - Answer: 307 customers (30.7%)
     | total_customers | churn_percentage |
     | --- | --- |
     | 307 | 30.7 |

5. **How many customers have churned straight after their initial free trial?**

   - SQL Code:
     ```sql
     WITH count_churn AS (
     SELECT *, LAG(plan_id, 1) OVER(PARTITION BY customer_id ) AS prev_plan
     FROM f_view
     )
     SELECT COUNT(prev_plan) AS total_churn,
     ROUND((COUNT(*)/ (SELECT COUNT(DISTINCT customer_id) FROM f_view)*100),0) AS percentage_churn
     FROM count_churn
     WHERE plan_id=4 AND prev_plan=0;
     ```
   - Answer: 92 customers (9%)
     | total_churn | percentage_churn |
     | --- | --- |
     | 92 | 9 |

6. **What is the number and percentage of customer plans after their initial free trial?**
   - SQL Code:
     ```sql
     WITH next_plan_table AS (
     SELECT *, LEAD(plan_id,1) OVER (PARTITION BY customer_id ORDER BY plan_id) AS next_plan
     FROM subscriptions
     )
     SELECT plan_name,
     COUNT(DISTINCT customer_id) AS total_customers,
     (100\*COUNT(DISTINCT customer_id)/
     (SELECT COUNT(DISTINCT customer_id) FROM f_view)) AS percentage_customers
     FROM next_plan_table t
     LEFT JOIN plans p
     ON p.plan_id =t.next_plan
     WHERE t.plan_id=0 AND
     next_plan is not null
     GROUP BY plan_name,next_plan
     ORDER BY total_customers;
     ```

- Answer:
  - Pro annual (37 customers, 3.7%)
  - Churn (92 customers, 9.2%)
  - Pro monthly (325 customers, 32.5%)
  - Basic monthly (546 customers, 54.6%)

| plan_name     | total_customers | percentage_customers |
| ------------- | --------------- | -------------------- |
| Pro annual    | 37              | 3.7%                 |
| Churn         | 92              | 9.2%                 |
| Pro monthly   | 325             | 32.5%                |
| Basic monthly | 546             | 54.6%                |

7. **Customer count and percentage breakdown of all 5 plan_name values at 2020-12-31.**

   - SQL Code:

     ```sql
     WITH CustomerCounts AS (
         SELECT
             p.plan_name,
             COUNT(DISTINCT s.customer_id) AS customer_count
         FROM plans p
         LEFT JOIN subscriptions s ON p.plan_id = s.plan_id
         WHERE s.start_date <= '2020-12-31'
         GROUP BY plan_name
     )

     SELECT
         plan_name,
         customer_count,
         ROUND((customer_count * 100) / (SELECT SUM(customer_count) FROM CustomerCounts), 2) AS percentage
     FROM CustomerCounts;
     ```

   - Answer:
     - Basic monthly: 538 customers (21.98%)
     - Churn: 236 customers (9.64%)
     - Pro annual: 195 customers (7.97%)
     - Pro monthly: 479 customers (19.57%)
     - Trial: 1000 customers (40.85%)

| plan_name     | customer_count | percentage |
| ------------- | -------------- | ---------- |
| Basic monthly | 538            | 21.98%     |
| Churn         | 236            | 9.64%      |
| Pro annual    | 195            | 7.97%      |
| Pro monthly   | 479            | 19.57%     |
| Trial         | 1000           | 40.85%     |

8. **How many customers have upgraded to an annual plan in 2020?**

   - SQL Code:
     ```sql
     SELECT COUNT(customer_id) AS total_customers
     FROM f_view
     WHERE start_date<="2020-12-31" AND
     plan_id=3;
     ```
   - Answer: 195 customers
     | total_customers |
     | --- |
     | 195 customers |

9. **How many days, on average, does it take for a customer to upgrade to an annual plan from their initial signup?**

   - SQL Code:
     ```sql
     WITH trial_plan_table AS
     (
     SELECT start_date as trial_date,customer_id
     FROM f_view
     WHERE plan_id=0
     ),
     annual_plan_table AS
     (
     SELECT start_date as annual_date,customer_id
     FROM f_view
     WHERE plan_id=3
     )
     SELECT
         ROUND(AVG(DATEDIFF(annual_date,trial_date)),0) as avg_days
     FROM trial_plan_table t
     JOIN annual_plan_table a
     ON a.customer_id=t.customer_id;
     ```
   - Answer: 105 days
     | avg_days |
     | --- |
     | 105 days |

10. **Average days for upgrading to an annual plan broken down into 30-day periods.**

- SQL Code:
  ```sql
  WITH trial_plan_table AS
  (
  SELECT start_date as trial_date,customer_id
  FROM f_view
  WHERE plan_id=0
  ),
  annual_plan_table AS
  (
  SELECT start_date as annual_date,customer_id
  FROM f_view
  WHERE plan_id=3
  )
  SELECT
      CONCAT(FLOOR(DATEDIFF( annual_date,trial_date) / 30) * 30, '-',
      FLOOR(DATEDIFF(annual_date,trial_date) / 30) * 30 + 30, ' days') AS period,
      COUNT(*) AS total_customers
  FROM trial_plan_table tp
  JOIN annual_plan_table ap ON tp.customer_id = ap.customer_id
  WHERE ap.annual_date IS NOT NULL
  GROUP BY CONCAT(FLOOR(DATEDIFF( annual_date,trial_date) / 30) * 30, '-',
      FLOOR(DATEDIFF(annual_date,trial_date) / 30) * 30 + 30, ' days')
  ORDER BY ROUND(AVG(DATEDIFF(annual_date,trial_date)), 0);
  ```

| period       | total_customers |
| ------------ | --------------- |
| 0-30 days    | 48              |
| 30-60 days   | 25              |
| 60-90 days   | 33              |
| 90-120 days  | 35              |
| 120-150 days | 43              |
| 150-180 days | 35              |
| 180-210 days | 27              |
| 210-240 days | 4               |
| 240-270 days | 5               |
| 270-300 days | 1               |
| 300-330 days | 1               |
| 330-360 days | 1               |

11. **How many customers downgraded from a pro monthly to a basic monthly plan in 2020?**
    
```sql
WITH next_plan_table AS
(
SELECT start_date,plan_id, LEAD(plan_id,1) OVER (PARTITION BY customer_id) AS next_plan
FROM f_view
)
SELECT COUNT(*) as total_customers
FROM next_plan_table
WHERE start_date <= '2020-12-31'
	AND plan_id = 2 AND next_plan = 1;
```

- Answer: No customers downgraded.

