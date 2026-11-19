---
layout: post
title: "Become user still works after that admin left"
date: 2026-11-19 12:00:00 +0100
permalink: /become-user-audit/
---

The admin account is suspended. HR closed the ticket. Canvas still has six months of Become User sessions in the page views, and two remaining root-account admins still have the permission.

Become User (masquerade) is an official Canvas admin action. It is not a bug. The leftover is the audit trail you never reconciled, and the root-account permission that outlived the person.

Inventory who can still Become User, and whether the masquerade events you expect to see still land in a store you watch.

## What Become User actually does

On self-hosted Canvas, an admin with the right permission can open a user and click **Become User**. Canvas starts a session as that user. The admin acts inside that user's privileges: sees what they see, submits what they can submit. A water-mark or a bar at the top of the page marks the masquerade in the browser. The LMS still treats the requests as that user's.

Instructure documents the permission as **Users - act as** (`become_user`). It is typically granted to Account Admin at the **root account**, not to Teacher. The permission is **not available at the subaccount level**. A sub-account admin can Become User only if they also hold a root-account role that includes it. Site admins with that root permission can Become User across the root account.

Canvas records masquerade in page views and, when Live Events are configured, as events you can subscribe to. Page views include the real admin and the user they became. If page views are rotated or the Canvas Data / Access Report export is off, the only evidence is the browser bar the admin already closed.

Run this on a test Canvas you own, or with a dedicated synthetic admin that never opens a real learner.

## Why the leftover shows up after someone leaves

Offboarding disables the admin login. It does not:

- revoke `become_user` on the *root-account role* that person used (the next person in Account Admin inherits it)
- expire historical masquerade page views
- tell you whether a shared break-glass admin was the actor
- prove that Become User events still reach the SIEM after a Data Services hiccup

A campus that treats “no current admin named X” as “nobody can Become User” is reading the user table and skipping the root-account role.

A campus that treats “we have page views” as “we could investigate a masquerade from March” is skipping retention: Canvas page view retention is an admin setting and a storage budget, not a legal hold.

## The fixture

On a test account:

1. List **root-account** roles that include **Users - act as** (`become_user`). Admin → Permissions (Account) at the root is the UI. The Account Roles API and Permission overrides are the machine path. Do not look for this permission on a subaccount role; Instructure does not expose it there.
2. List users currently in those root-account roles.
3. Create a synthetic student `masq-canary-001` and a synthetic admin `masq-admin-001` that has Become User on a root-account role. Do not use a production admin.
4. As `masq-admin-001`, Become User as `masq-canary-001`, open one canary page, Stop Acting as User.
5. Read three stores for that interval: page views for the student, page views for the admin, and (if you run it) the Live Events stream.

```bash
# only against a Canvas you own
AUTH="Authorization: Bearer $CANVAS_TOKEN"

# users currently able to act-as: start from root account admins
curl -sS -H "$AUTH" \
  "$CANVAS/api/v1/accounts/self/admins" \
  -o /tmp/canvas-admins.json

# page views for the canary user after the masquerade
curl -sS -H "$AUTH" \
  "$CANVAS/api/v1/users/sis_user_id:masq-canary-001/page_views?per_page=20" \
  -o /tmp/canvas-canary-views.json
```

Keep a row per run:

| Field | Value |
| --- | --- |
| Admin user id | |
| Target user id | |
| Time Become User started / stopped | |
| Page view ids that show the masquerade | |
| Live Event received (yes/no) | |
| Root-account roles that still have `become_user` | |

Alert when:

- A user in a root-account `become_user` role is suspended, deleted, or past their last day, and the role still has the permission for everyone else with no break-glass procedure.
- The masquerade you just generated has no page view within the documented delay.
- Page view retention is shorter than the investigation window you claim in the security policy.
- `become_user` is enabled on a root-account role that more people sit in than the named break-glass list.

## What this inventory is not

This is not a ban on Become User. Support organisations use it. The control is: a named list of people on a root-account role, a store that still has the event, and a revoke step that is part of admin offboarding — the user *and* a review of who else sits in that role.

It is a different check from leftover OAuth developer keys and leftover SIS files. Those are credentials and uploads. This is a privileged session type.

Disclosure: I work on [GuardLMS](https://guardlms.com/en/integrations/canvas), a product of Ldesign Media. Become User, the `become_user` permission (root-account only), and page views are Canvas admin behaviour on self-hosted instances; confirm the permission label on your release before you write the alert.

