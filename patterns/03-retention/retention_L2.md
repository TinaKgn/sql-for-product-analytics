# 1. Retention rate by users' sign-up month

**Pattern:** Retention rate \
**Product:** NorthStar | B2B SaaS \
**Level of difficulty:** L2

## Business question
Of users who signed up in a given month, what percentage fired create_project at any point during their second calendar month?
Return signup_cohort, cohort_size, retained_users, retention_rate.


## Metrics definition
**Grain:** user \
**Retained users:** The number of distinct users in a signup cohort who performed the defined activity during the specified return period. \
**Retention rate:** The percentage of users in a cohort who return and perform a defined activity during a specified time period.

## Assumptions
For the first version, let's assume that every signed up user has at least one record in the events table due to the 'signup' event.

The second version is optimised for non-completeness assumption (missing signup values in the events table, so that some users don't have any record there) and for data processing speed.

## Solution logic and heuristics

### Solution 1.

The solution below has 3 steps: we first assign each user to a cohort labeled by their signup month, then count the users per cohort who have a `create_project` event in the month following their signup month (this is our definition of "retained"), and finally calculate the retention rate as the ratio of retained users to cohort size.

Since we assumed here that every user has a record in `events`, we use an `INNER JOIN`. This logic can create multiple rows per user, since one user can create multiple projects. This is solved by `COUNT(DISTINCT ...)`.

```sql
WITH assigned_cohort AS (
    SELECT user_id,
    DATE_TRUNC('month',signup_date) AS signup_cohort
    FROM users
    ),

created_project AS (
    SELECT signup_cohort,
    COUNT (DISTINCT ac.user_id) AS cohort_size,
    COUNT (DISTINCT CASE WHEN e.event_name = 'create_project'
    AND DATE_TRUNC('month', e.event_date) = ac.signup_cohort + INTERVAL '1 month' THEN ac.user_id END) AS retained_users
    FROM assigned_cohort ac
    JOIN events e
    ON ac.user_id = e.user_id
    GROUP BY signup_cohort
    )

SELECT signup_cohort,
cohort_size,
retained_users,
CAST (retained_users AS decimal) / cohort_size * 100 AS retention_rate
FROM created_project
```

### Solution 2.

Let's assume the data is incomplete, and a signed-up user can have zero records in the `events` table. Those users would be lost after the `INNER JOIN`.

Another thing to consider is that the previous solution has a performance issue: `JOIN` + `COUNT(DISTINCT CASE WHEN ...)` physically creates every matching combination (user, event) first. All of a user's events get joined in, not just the qualifying ones. Then it collapses them back down via `DISTINCT`. On a large `events` table, this means more rows are processed before the dedup happens, which can be meaningfully slower at scale.

`EXISTS` in a subquery, instead, stops looking the moment it finds one matching row per user, rather than materializing every matching row. Since this version does not join `events` in the outer query, users with zero records in `events` still count towards `cohort_size`.


```sql
WITH assigned_cohort AS (
    SELECT user_id,
    DATE_TRUNC('month',signup_date) AS signup_cohort
    FROM users
    ),

created_project AS (
    SELECT ac.signup_cohort,
    COUNT (DISTINCT ac.user_id) AS cohort_size,
    COUNT (DISTINCT CASE WHEN EXISTS (
            SELECT 1
            FROM events e
            WHERE e.event_name = 'create_project'
            AND DATE_TRUNC('month', e.event_date) = ac.signup_cohort + INTERVAL '1 month'
            AND e.user_id = ac.user_id
            )
            THEN ac.user_id END) AS retained_users
    FROM assigned_cohort ac
    GROUP BY ac.signup_cohort
    )

SELECT signup_cohort,
cohort_size,
retained_users,
CAST (retained_users AS decimal) / cohort_size * 100 AS retention_rate
FROM created_project
```

## Potential trap

Note that the `EXISTS` subquery must have a line linking the condition (a `create_project` event in the target month) to the specific user (`AND e.user_id = ac.user_id`). Without it, `EXISTS` isn't asking "did this user do it", it's asking "did someone, somewhere do it". As soon as any single user creates a project in the target month, every user in that cohort would be marked as retained, wildly inflating `retained_users`.