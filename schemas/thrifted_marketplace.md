# Thrifted | Marketplace

## Business context

Thrifted is a peer-to-peer marketplace for secondhand clothing and goods. Sellers list items, and buyers browse and order directly from individual sellers.

Thrifted's ops and growth teams care most about GMV (gross merchandise value) by category and seller, repeat seller activity (does someone who lists once come back and list again), and order completion (do listings actually convert into completed orders, or stall). Reviews are a secondary trust signal, useful for spotting problem categories or sellers, but not the primary health metric.

## Conventions

- Dialect: PostgreSQL.
- Timestamps are stored in UTC. `listing_events.event_date` keeps its original name but is a `TIMESTAMP`; every other `*_date` column is a plain `DATE`.
- A listing is a single physical item, so at most one completed order is expected per `listing_id`. A second order on the same listing would be a data anomaly, not normal behaviour.
- Order value is already stored per order (`orders.order_value`); there is no separate revenue table.
- `listing_events` is scoped to listings and has no `user_id`. User-level activity on Thrifted is derived from `orders`, `reviews`, `listings` and `referrals`.
- System-generated events (`price_drop_alert_sent`, `listing_expired`) are ordinary rows in `listing_events`. Whether they count in a funnel or a sequence is decided in each challenge, not in the schema.

## Schema

```sql
CREATE TABLE users (
    user_id       INTEGER PRIMARY KEY,
    signup_date   DATE NOT NULL,
    user_country  TEXT NOT NULL
);

CREATE TABLE listings (
    listing_id  INTEGER PRIMARY KEY,
    seller_id   INTEGER NOT NULL REFERENCES users (user_id),
    category    TEXT NOT NULL,
    price       NUMERIC(10, 2) NOT NULL
);

CREATE TABLE orders (
    order_id     INTEGER PRIMARY KEY,
    buyer_id     INTEGER NOT NULL REFERENCES users (user_id),
    listing_id   INTEGER NOT NULL REFERENCES listings (listing_id),
    order_date   DATE NOT NULL,
    order_value  NUMERIC(10, 2) NOT NULL
);

CREATE TABLE reviews (
    review_id    INTEGER PRIMARY KEY,
    order_id     INTEGER NOT NULL REFERENCES orders (order_id),
    rating       SMALLINT NOT NULL CHECK (rating BETWEEN 1 AND 5),
    review_date  DATE NOT NULL
);

CREATE TABLE listing_events (
    event_id    INTEGER PRIMARY KEY,
    listing_id  INTEGER NOT NULL REFERENCES listings (listing_id),
    event_name  TEXT NOT NULL CHECK (event_name IN (
                    'view_listing', 'add_to_cart', 'order_placed', 'review_left',
                    'price_drop_alert_sent', 'listing_expired')),
    event_date  TIMESTAMP NOT NULL
);

CREATE TABLE referrals (
    referrer_user_id  INTEGER NOT NULL REFERENCES users (user_id),
    referred_user_id  INTEGER REFERENCES users (user_id),   -- NULL if the invite never converted
    invited_at        TIMESTAMP NOT NULL,
    signed_up_at      TIMESTAMP,                            -- NULL if the invite never converted
    CHECK ((referred_user_id IS NULL) = (signed_up_at IS NULL))
);
```

## Notes

**Event types in `listing_events`**

| `event_name` | Kind | Meaning |
|---|---|---|
| `view_listing` | buyer | A buyer opens the listing |
| `add_to_cart` | buyer | A buyer adds the listing to their cart |
| `order_placed` | buyer | A buyer places an order on the listing |
| `review_left` | buyer | A review is left for the order |
| `price_drop_alert_sent` | system | Product notifies interested buyers of a price drop |
| `listing_expired` | system | Listing is automatically taken down after a period without a sale |

**Column rules**

- `orders` is the system of record for what was actually bought. `listing_events.order_placed` is the logged action; both should exist for a real order, but they are not interchangeable.
- `referrals` has one row per invite sent. Invites that never converted have `referred_user_id` and `signed_up_at` both `NULL`; they stay in the table.

## Version history

- **v1:** `users`, `listings`, `orders`, `reviews`.
- **v2:** added `listing_events` to support multi-stage funnel challenges.
- **v3:** added `referrals`; new event types `price_drop_alert_sent` and `listing_expired`; `listing_events.event_date` declared as `TIMESTAMP`. No existing column was renamed or removed, so solutions written against v1 or v2 still run unchanged.