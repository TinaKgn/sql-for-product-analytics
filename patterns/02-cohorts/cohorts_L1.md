# 1. Users' cohorts by month of program start

**Pattern:** Cohorts \
**Product:** 'Wellspring' | Consumer Health App \
**Level of difficulty:** L1

## Business question

Assign each user to a cohort by the month of their start_program event. Return each cohort month with its cohort_size.

## Assumptions
At this level of difficulty, let's assume that each user starts at most one program.

## Solution heuristic

DATE_TRUNC to bucket users by month, then GROUP BY cohort.

## Metric definition

**Grain:** user

## Solution

```sql
SELECT COUNT (DISTINCT user_id) AS cohort_size,
DATE_TRUNC('month',event_date) AS start_program_month
FROM events
WHERE event_name = 'start_program'
GROUP by start_program_month
```
## Why this works
Since we assumed that one user starts a program only once, we simply need to find out the months of each distinct user's start_program event and then group (count) users by months.