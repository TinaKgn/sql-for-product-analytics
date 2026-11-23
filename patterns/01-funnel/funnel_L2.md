# 1. Listings with a completed order

**Pattern:** Funnel analysis \
**Product:** 'Thrifted' Marketplace \
**Level of difficulty:** L2

## Business question
Of all listings created, how many went on to be sold (i.e., have at least one order placed against that listing_id)?
Return a single count: the number of distinct listings with a completed order.

## Assumptions

The assumption here is that order_id is a reliable, never-null primary key on the orders table.
We also expect every order to have both a **listing_events** record ('order_placed') and a matching **orders** record.

## Solution heuristic

Think about it as the simplest possible funnel: one step, one conversion point (listing created → order placed).

## Metric choice and definition

**Grain:** listing

We have two relevant metrics available in our data schema: 'order_placed' in **listing_events** as a logged event (a timestamped action) and actual transaction records in **orders** (with buyer_id, order_value, etc.).
For the purpose of this task, we'll look at the **orders** dataset: it's enough for us to know that a listing appears there. If we chose to look in the  **listing_events** instead, we'd need to run an additional check (listing_events.event_name = 'order_placed').

## Solution

```sql
SELECT COUNT (DISTINCT l.listing_id) as sold_listings
FROM listings l
    WHERE EXISTS (
        SELECT 1
        FROM orders o
        WHERE o.listing_id = l.listing_id
        )
```
## Why this works

- `EXISTS` is a correlated subquery: for each `listing_id` in `listings`, it checks whether at least
  one matching row exists in `orders` — without caring how many matches there are.
- `COUNT(DISTINCT l.listing_id)` then counts each qualifying listing once, even if (hypothetically) it
  had more than one order.
- No join is used in the outer query, so there's no row-multiplication risk. `EXISTS` only returns true/false per listing.