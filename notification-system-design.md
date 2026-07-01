# Notification System Design
# Stage 1

## Notification_System_Design.md

### The problem

Users miss important notifications because they're buried in a busy
feed. We need to surface the top-n most important ones, based on:

- **Weight** — type matters: `Placement > Result > Event`
- **Recency** — newer notifications should generally rank higher

### How scoring works

Each notification gets one score between 0 and 1:

```
score = 0.7 * typeScore + 0.3 * recencyScore
```

- `typeScore` — Placement=3, Result=2, Event=1, normalized to 0–1
- `recencyScore` — exponential decay, halves every 6 hours, so a
  notification from right now scores ~1 and fades smoothly over time

Type is weighted more heavily than recency (0.7 vs 0.3), so a Placement
notification from a few hours ago still beats a brand-new Event — but
an old Result will eventually drop below a fresh Event as it decays.

Ties are broken by timestamp (most recent wins).

### Why decay instead of buckets like "today / this week / older"?

Buckets create hard cutoffs — a notification 1 minute past midnight
suddenly drops a whole tier. A decay curve ages notifications smoothly
and is easy to tune (just change the half-life or the weight split)
without rewriting logic.

### No database, per the constraints

Notifications aren't stored anywhere. `priorityNotifications.js` pulls
the current list straight from the evaluation API on each call and
scores/sorts it in memory.

### Keeping "top N" efficient as new notifications come in

- At the current feed size, recomputing the full sort on every request
  is cheap (O(m log m) on maybe a few hundred items) — no need to
  over-engineer this yet.
- If the feed grew much larger, the efficient move is a bounded
  min-heap of size N: for each incoming notification, compare it to
  the heap's current minimum and swap in O(log n) instead of re-sorting
  everything.
- Because scores decay with time, "top N" shifts even with zero new
  notifications — so recomputing on read (rather than caching a fixed
  "correct" top N) is the right model here. The frontend just re-polls
  periodically to stay fresh.

### Logging

`log()` in `priorityNotifications.js` writes a structured JSON line for
every fetch, so you can see request timing and result counts in the
console. The frontend (`notification-app-fe/src/logger.js`) mirrors the
same shape so logs read consistently across both tiers.
