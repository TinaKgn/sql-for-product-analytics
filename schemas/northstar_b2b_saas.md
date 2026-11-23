# NorthStar | B2B SaaS

## Business context

NorthStar is a project- and task-management tool sold to companies. Teams sign up, create projects, invite teammates and track tasks through to completion. The model is freemium: free accounts hit limits quickly, while paid accounts unlock unlimited projects and teammates.

NorthStar's growth team cares about three things: activation (does a new signup set up a project, invite someone and get work done, or quietly drift away), expansion (do teams upgrade as they grow), and account value (which accounts are worth the most and which are at risk).

## Conventions

- Dialect: PostgreSQL.
- Timestamps are stored in UTC. `events.event_date` keeps its original name but is a `TIMESTAMP`; every other `*_date` column is a plain `DATE`.
- **Activated user:** a user with at least one `complete_task` event (first real value).
- `company_size` is the coarse bucket of the account's `employee_band`: small = 1-50, medium = 51-500, large = 501+.
- Several users can belong to one account (`users.account_id`).
- System-generated events (`notification_sent`, `weekly_digest_sent`) are ordinary rows in `events`. Whether they count as user activity is decided in each challenge, not in the schema.

## Schema

```sql
CREATE TABLE accounts (
    account_id      INTEGER PRIMARY KEY,
    employee_band   TEXT NOT NULL
                    CHECK (employee_band IN ('1-10', '11-50', '51-200', '201-500', '501+')),
    contract_value  NUMERIC(12, 2)      -- annual contract value in EUR; NULL = no contract (free)
);

CREATE TABLE users (
    user_id       INTEGER PRIMARY KEY,
    signup_date   DATE NOT NULL,
    company_size  TEXT NOT NULL CHECK (company_size IN ('small', 'medium', 'large')),
    plan_type     TEXT NOT NULL CHECK (plan_type IN ('free', 'paid')),
    account_id    INTEGER NOT NULL REFERENCES accounts (account_id)
);

CREATE TABLE features (
    feature_id    INTEGER PRIMARY KEY,
    feature_name  TEXT NOT NULL UNIQUE,
    tier          TEXT NOT NULL CHECK (tier IN ('core', 'key', 'advanced'))
);

CREATE TABLE events (
    event_id      INTEGER PRIMARY KEY,
    user_id       INTEGER NOT NULL REFERENCES users (user_id),
    event_name    TEXT NOT NULL CHECK (event_name IN (
                      'signup', 'create_project', 'invite_teammate', 'create_task',
                      'complete_task', 'upgrade_plan', 'use_feature', 'invite_sent',
                      'notification_sent', 'weekly_digest_sent')),
    event_date    TIMESTAMP NOT NULL,
    project_id    INTEGER,              -- see notes below
    feature_name  TEXT REFERENCES features (feature_name)   -- only for use_feature
);

CREATE TABLE subscriptions (
    subscription_id  INTEGER PRIMARY KEY,
    user_id          INTEGER NOT NULL REFERENCES users (user_id),
    plan             TEXT NOT NULL,
    start_date       DATE NOT NULL,
    monthly_price    NUMERIC(10, 2) NOT NULL
);

CREATE TABLE referrals (
    referrer_user_id  INTEGER NOT NULL REFERENCES users (user_id),
    referred_user_id  INTEGER REFERENCES users (user_id),   -- NULL if the invite never converted
    invited_at        TIMESTAMP NOT NULL,
    signed_up_at      TIMESTAMP,                            -- NULL if the invite never converted
    CHECK ((referred_user_id IS NULL) = (signed_up_at IS NULL))
);

CREATE TABLE nps_responses (
    user_id       INTEGER NOT NULL REFERENCES users (user_id),
    score         SMALLINT NOT NULL CHECK (score BETWEEN 0 AND 10),
    responded_at  TIMESTAMP NOT NULL,
    PRIMARY KEY (user_id, responded_at)
);
```

## Notes

**Event types**

| `event_name` | Kind | Meaning |
|---|---|---|
| `signup` | user | Account created |
| `create_project` | project | User creates a project |
| `invite_teammate` | project | User adds an existing or new teammate to a specific project |
| `create_task` | project | User creates a task |
| `complete_task` | project | User completes a task |
| `upgrade_plan` | user | User moves from free to paid |
| `use_feature` | usage | User uses a product feature; `feature_name` is populated |
| `invite_sent` | user | User invites someone to join NorthStar itself; feeds `referrals` |
| `notification_sent` | system | Product sent a notification to the user |
| `weekly_digest_sent` | system | Product sent the weekly digest email |

**Column rules**

- `project_id` is populated for `create_project`, `invite_teammate`, `create_task` and `complete_task` (and may be populated for `use_feature`). It is `NULL` for user-level and system events.
- `feature_name` is populated only for `use_feature` events.
- `referrals` has one row per invite sent. Invites that never converted have `referred_user_id` and `signed_up_at` both `NULL`; they stay in the table.
- Only a subset of users answer the NPS survey, so `nps_responses` does not cover every user.

## Version history

- **v1:** `users`, `events` (with `project_id`), `subscriptions`.
- **v2:** added `accounts`, `features`, `referrals`, `nps_responses`; `users.account_id`; `events.feature_name`; new event types `use_feature`, `invite_sent`, `notification_sent`, `weekly_digest_sent`; `event_date` declared as `TIMESTAMP`. No existing column was renamed or removed, so solutions written against v1 still run unchanged.