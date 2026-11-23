# 1. Users' cohorts by month of program start

**Pattern:** Cohorts \
**Product:** 'Wellspring' | Consumer Health App \
**Level of difficulty:** L1

## Business question
Assign each user to a cohort by the month of their start_program event. Return each cohort month with its cohort_size.

## Assumptions
Let's assume here that a user can have multiple programs, i.e. more than one start_program event per user.

## Solution logic and heuristics

1. Find out all the start_program months for each user.
2. Assign a cohort by selecting the very first start_program month for each user.
3. Group by start program months and count unique users.


## Metric definition

**Grain:** user

A data-modeling judgment call: what does "cohort" mean when a user can belong to more than one program-start? Is the cohort based on the user's first-ever start_program across all their programs, or could a user legitimately belong to two different cohort months if they started a second program later?

For this challenge, let's go with the standard definition — first-ever start_program per user.

## Solution

```sql
WITH cohort_month AS (
    SELECT user_id,
    MIN (DATE_TRUNC('month',event_date)) AS first_program_month
    FROM events
    WHERE event_name = 'start_program'
    GROUP BY user_id
    )

SELECT first_program_month,
COUNT (DISTINCT user_id) AS cohort_size
FROM cohort_month
GROUP BY first_program_month
```
## Why this works

A simple single-stage GROUP BY distinct users by start months would produce a wrong answer here or fail, considering that one user could belong to two different months.

## Alternative solutions

Some tasks where the first record has to be find, can be solved with ROW_NUMBER() function. However, ROW_NUMBER solves a related but slightly different problem:
"give me the whole row corresponding to the earliest event" (useful when you need other columns from that specific row). Here, we don't actually need anything else from the start_program row besides its date.
Since the date alone is enough to assign a cohort month, MIN + WHERE is simpler and avoids an unnecessary CTE layer.
If you find yourself needing some other column tied to that specific first-program row, that's when ROW_NUMBER earns its place.