---
layout: post
title: "When the LMS security log goes silent"
date: 2026-11-18 12:00:00 +0100
permalink: /log-heartbeat/
---

A quiet SIEM can mean a quiet campus. It can also mean the LMS stopped writing events, the forwarder dropped them, or the subscription was deactivated after a weekend of 5xx responses. The cheapest way to tell those apart is a heartbeat: one synthetic account, one documented benign event, and a reconcile of that event’s id and timestamps across the LMS store, the forwarding queue, and the SIEM.

Run this on a test instance, or with a dedicated synthetic user that never touches learner data. The event is a login or a view of the canary course from [the authorization-fixture article]({{ site.baseurl }}/canary-fixtures/). It is not a grade change on a live student.

## What the LMS actually stores

Moodle™ writes to the standard log table `logstore_standard_log`. The columns that matter for a heartbeat are `id`, `eventname`, `userid`, `courseid`, `objectid`, `timecreated`, `origin`, and `ip`. `origin` is a short string such as `web`, `cli`, `ws`, or `cron`. The login event is `\core\event\user_loggedin` (`crud = r`, object table `user`). A course home view is `\core\event\course_viewed` (`crud = r`, context level course).

The store has four admin settings under Site administration → Plugins → Logging → Standard log:

- **Log guest actions** (`logstore_standard/logguests`), default on.
- **JSON format** for the `other` field, default on in current 4.5.
- **Keep logs for** (`logstore_standard/loglifetime`): never delete, or 2, 5, 10, 35, 60, 90, 120, 150, 180, 365, or 1000 days. Cron deletes older rows.
- **Write buffer size** (`logstore_standard/buffersize`), default 50.

Events are observed after the database transaction commits. A row can land a few seconds after the click. A buffer of 50 means the last handful of events in a burst may sit in PHP until the buffer flushes. Site administration → Reports → Logs is the UI over the same table.

Canvas does not give you that table. It emits Live Events from Data Services: Admin → Data Services, after the Canvas Data Services developer key is installed and the app is added on Account Settings. Delivery is AWS SQS or an HTTPS webhook. The queue name must begin with `canvas-live-events`. The SQS policy must allow account `636161780776`. Message format is Canvas JSON or Caliper 1.1. Live Events run in Production and Beta. They are for analytics pipelines; Instructure’s own introduction says to use the REST API when you need the most current data.

Two event names work as heartbeats. `logged_in` is emitted when a user logs in; the payload has `metadata.event_name`, `metadata.event_time`, `metadata.user_id`, `metadata.request_id`, and `body.redirect_url`. `asset_accessed` is emitted when a user views an object; the body has `asset_type`, `asset_id`, and `asset_subtype`. Subscribe to one of those, not to every event on the account, until the pipeline is proven.

HTTPS delivery expects a 2xx within 15 seconds. Redirects do not count as success. Failed deliveries are retried six times, seven attempts in total, with exponential backoff and jitter: the last retry is roughly 32 to 96 minutes after the first failure. Handlers must be idempotent; a timeout can still deliver the same event twice. Delivery is unordered. After retries are exhausted, Canvas records the event as failed and stops. An endpoint that keeps failing is deactivated at about 9,500 timeout errors or 28,000 other errors in 24 hours, and the subscription owner gets email. Canvas will not tell you which event ids were dropped beyond a count and a status class.

## Generate one event

Moodle™:

```bash
# 1. log in as the synthetic user (or view the canary course)
# 2. wait for the logstore buffer, then query by user and event name
# prefix and table name follow your install; default table is mdl_logstore_standard_log

psql -c "
SELECT id, eventname, userid, courseid, objectid,
       to_timestamp(timecreated) AS ts, origin, ip
FROM mdl_logstore_standard_log
WHERE userid = :canary_user_id
  AND eventname = '\\core\\event\\user_loggedin'
  AND timecreated >= extract(epoch from now() - interval '15 minutes')
ORDER BY id DESC
LIMIT 5;
"
```

Record `id`, `timecreated`, `origin`, and `ip`. That `id` is the heartbeat token for this run.

Canvas:

Log in as the synthetic user, or `GET` the canary page with that user’s token so an `asset_accessed` fires. On the HTTPS endpoint, log `metadata.request_id`, `metadata.event_name`, `metadata.event_time`, and `metadata.user_id`. On SQS, the same fields are in the message body. The `request_id` is the join key; there is no Moodle-style monotonic `id` in the Canvas JSON.

## Reconcile three clocks

For each run, write one row:

| Field | LMS | Forwarder | SIEM |
| --- | --- | --- | --- |
| Event id / request id | | | |
| Event name | | | |
| User id | | | |
| LMS `timecreated` / `event_time` | | | |
| Time the forwarder accepted it | | | |
| Time the SIEM indexed it | | | |
| Duplicate copies | | | |

Alert when:

- The LMS has the row and the forwarder does not, after the documented retry window (for Canvas HTTPS, wait through the last retry, about 96 minutes, before calling it lost).
- The forwarder has it and the SIEM does not.
- The SIEM has it and the LMS does not: a duplicate or a replay.
- `origin` on Moodle™ is `cli` or `ws` for a heartbeat you generated in the browser.
- Canvas `event_time` and SIEM `@timestamp` drift by more than your SLA. Live Events timestamps are ISO 8601 with an offset; parse the offset.
- Moodle™ `loglifetime` is 10 days and the SIEM retention is 90: the LMS will go silent for historical queries while the SIEM still answers. That is a retention gap, not a pipeline outage.
- The Canvas subscription shows Inactive. Inactive streams are kept for 90 days, then they disappear from the list.

A missing heartbeat is the finding. A late heartbeat is a latency SLO. A duplicated `request_id` is an idempotency bug in the handler; Canvas documents that a slow 2xx can still be retried.

## What silence is not

Do not treat “no `logged_in` events this hour” as proof that nobody signed in. Canvas Live Events skip events when the destination is down, and those events are not replayed after the seven attempts. Moodle™ will stop inserting when the standard store is disabled under Manage log stores, or when the disk under the table is full. `logguests` off will drop guest traffic and look like a quiet public course.

Keep the synthetic user on an allow-list so a lockout policy does not kill the probe. Rotate its password in the same vault as the other monitoring secrets. Redact `ip` and `user_agent` before the row leaves the security team, if the SIEM is wider than that team.

The heartbeat is observability of the audit pipeline. It does not replace an LTI inventory, a canary authorization fixture, or a header check. It answers one question: if this identity does one documented thing, does the evidence of that thing still arrive, today, in every store you claim to watch.

Disclosure: I work on [GuardLMS](https://guardlms.com). The event names, table columns, and retry limits above are from the platforms’ own documentation.
