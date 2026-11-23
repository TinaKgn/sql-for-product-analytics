# Wellspring | Consumer Health App

## Business context

Wellspring is a consumer digital health app. Patients are prescribed a program (physio, mental health, chronic condition management) by a clinician, and work through it with guided exercise sessions in the app, with human coach check-ins layered on top.

Wellspring's product and clinical teams care about adherence (do patients actually complete prescribed sessions, or drop off early), onboarding effectiveness (does a new intro flow improve week-2 engagement), and intervention testing (does a reminder nudge or a coach check-in actually move completion rates). There is no revenue per user in the data, so value is measured through key actions instead.

## Conventions

- Dialect: PostgreSQL.
- Timestamps are stored in UTC. `events.event_date` keeps its original name but is a `TIMESTAMP`; `users.signup_date` and `experiment_assignment.assigned_date` are plain `DATE`s.
- A user can have multiple programs, i.e. more than one `start_program` event per user.
- **Activated user:** a user with at least one `complete_exercise_session` event (first real value).
- **Key actions** (the value proxy, in place of revenue): `complete_exercise_session`, `complete_assessment`, `message_coach`.
- System-generated events (`notification_sent`, `reminder_sent`) are ordinary rows in `events` with `session_id = NULL`. Whether they count as user activity is decided in each challenge, not in the schema.

## Schema

```sql
CREATE TABLE users (
    user_id      INTEGER PRIMARY KEY,
    signup_date  DATE NOT NULL
);

CREATE TABLE sessions (
    session_id     INTEGER PRIMARY KEY,
    user_id        INTEGER NOT NULL REFERENCES users (user_id),
    session_start  TIMESTAMP NOT NULL,
    session_end    TIMESTAMP NOT NULL,
    device_type    TEXT NOT NULL,        -- 'mobile', 'desktop' or 'tablet'
    action_count   INTEGER NOT NULL DEFAULT 0,
    CHECK (session_end >= session_start)
);

CREATE TABLE events (
    event_id    INTEGER PRIMARY KEY,
    user_id     INTEGER NOT NULL REFERENCES users (user_id),
    session_id  INTEGER REFERENCES sessions (session_id),   -- NULL for system-generated events
    event_name  TEXT NOT NULL CHECK (event_name IN (
                    'start_program', 'complete_exercise_session', 'skip_session',
                    'message_coach', 'renew_subscription', 'complete_assessment',
                    'invite_sent', 'share_progress',
                    'notification_sent', 'reminder_sent')),
    event_date  TIMESTAMP NOT NULL
);

CREATE TABLE experiment_assignment (
    user_id          INTEGER NOT NULL REFERENCES users (user_id),
    experiment_name  TEXT NOT NULL,
    variant          TEXT NOT NULL CHECK (variant IN ('control', 'treatment')),
    assigned_date    DATE NOT NULL,
    PRIMARY KEY (user_id, experiment_name)
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
| `start_program` | user | Patient starts a prescribed program (a user can start several over time) |
| `complete_exercise_session` | key action | Patient completes a guided exercise session |
| `skip_session` | user | Patient skips a scheduled session |
| `message_coach` | key action | Patient messages their coach |
| `renew_subscription` | billing | Patient renews their subscription |
| `complete_assessment` | key action | Patient completes a clinical check-in assessment |
| `invite_sent` | user | Patient invites someone to Wellspring; feeds `referrals` |
| `share_progress` | user | Patient shares their progress outside the app |
| `notification_sent` | system | Product sends a push notification |
| `reminder_sent` | system | Product sends a session reminder |

**Column rules**

- `sessions.device_type` takes the values `'mobile'`, `'desktop'` or `'tablet'`.
- `sessions.action_count` is the number of in-app actions during the session.
- `referrals` has one row per invite sent. Invites that never converted have `referred_user_id` and `signed_up_at` both `NULL`; they stay in the table.
- Only a subset of users answer the NPS survey, so `nps_responses` does not cover every user.

## Version history

- **v1:** `sessions` (`session_id`, `user_id`, `session_start`, `device_type`), `events`, `experiment_assignment`.
- **v2** added `users`, `referrals`, `nps_responses`; `sessions.session_end` and `sessions.action_count`; new event types `complete_assessment`, `invite_sent`, `share_progress`, `notification_sent`, `reminder_sent`; `event_date` declared as `TIMESTAMP`.