# 1. Retention rate by users' sign-up month

**Pattern:** Top-N per group \
**Product:** Thrifted | Marketplace \
**Level of difficulty:** L1

## Business question
For each category, find the single highest-priced listing. Return category, listing_id, price.

## Assumption
Let's assume that no two listings in the same category share the highest price. Tie handling is covered in the L2 challenges of this pattern.

## Solution logic and heuristics

**Recognise this pattern from:** most popular per..., best-selling within each..., highest per...

```sql
WITH ranked_categories AS (
    SELECT category,
    listing_id,
    price,
    ROW_NUMBER() OVER (PARTITION BY category ORDER BY price DESC) AS rn
    FROM listings
)

SELECT category,
listing_id,
price
FROM ranked_categories
WHERE rn = 1;
```

## Possible traps

`ROW_NUMBER()` always gives exactly one row per group when you filter rn = 1. That matches questions like "the most recent page per user" or "the single highest-priced listing per category". It also extends cleanly: rn <= 3 gives exactly three rows per group. In case of a tie, `ROW_NUMBER()` silently picks one arbitrarily. `RANK()` would show both as rank 1, so you get more than one row for that category.

Which is right depends on the business question. "The single highest-priced listing" wants one row, so ROW_NUMBER, ideally with a tiebreaker in the `ORDER BY` (e.g. price DESC, listing_id) so the result is deterministic. "The highest-priced listing(s)" wants all tied rows, so RANK.

The solution above has this weakness: two equally priced listings in a category would return an arbitrary one of them.