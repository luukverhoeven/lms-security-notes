---
layout: post
title: "The WordPress application password survived the password reset"
date: 2026-11-19 12:00:00 +0100
permalink: /wp-application-passwords/
---

The staff member reset their WordPress password on the last day. The REST client they installed in March still authenticates. Application Passwords are a different credential from the login password.

WordPress 5.6 shipped Application Passwords for the REST API (and XML-RPC) over HTTPS. They are created on the user profile, shown once, stored hashed, and revoked individually. A main-password reset is not a revoke.

Inventory Application Passwords the way you inventory leftover tokens: one row per password, against people who still work there.

## What an Application Password actually is

On wp-admin → Users → Profile (or Edit User), **Application Passwords** issues a 24-character secret. WordPress shows it once. After that you only see the name, a truncated hint, the creation date, and last used.

The client authenticates to `/wp-json/` with HTTP Basic over HTTPS:

```bash
# only against a WordPress you own
curl -sS --user "USERNAME:APPLICATION_PASSWORD" \
  "$HOST/wp-json/wp/v2/users/me?context=edit"
```

A 200 with that user’s JSON means the Application Password still works. A 401 means it was revoked or never existed. Cookie + `X-WP-Nonce` is a different mechanism (browser sessions) and is not this leftover.

WordPress hashes Application Passwords; the plaintext is not in the database. Revoke is a button on that same profile, per password or all at once. There is no `validuntil` column in core.

They also work on `xmlrpc.php` when XML-RPC is enabled. That is a second path for the same secret. Leftover `xmlrpc.php` is a separate note; do not collapse the two.

Run this only against a WordPress you own.

## Why it outlives the password reset

Offboarding playbooks reset or expire the WordPress login password, disable the user, or delete the user. Those three are not equal:

- **Password reset** — the login password changes. Application Passwords on that user remain until revoked. The March CI bot still has a working Basic header.
- **Disable / set role to none** — REST capability checks start failing for privileged routes. `users/me` may still 200. Treat “role none” as incomplete offboarding until the Application Password list is empty.
- **Delete user** — the user row goes away and the Application Passwords go with it. Reassign content first. Deletion is the only one of the three that is a real token lifecycle by default.

A contractor who “just needed REST for a report” will have a password named `Insomnia` or `curl` with a last-used date after their contract ended if nobody opened that profile.

## The fixture

On a test WordPress you own:

1. Create user `apppw-canary-001` with an editor role.
2. Issue an Application Password named `canary-rest-001`. Store it in the vault. Confirm `GET /wp-json/wp/v2/users/me` works with Basic auth.
3. Reset the user’s *login* password. Repeat the REST call. Expect 200 if you have not revoked the Application Password — that is the leftover this check exists to catch.
4. Revoke the Application Password on the profile. Repeat. Expect 401.
5. For production inventory, export every user with at least one Application Password: user id, login, email, role, application name, created, last used.

```bash
# WP-CLI on a host you own
wp user list --fields=ID,user_login,user_email,roles
wp user application-password list USERLOGIN --format=table
```

Alert when:

- An Application Password exists for a user whose last login is older than your offboarding SLA, or whose email is a leaver.
- Last used is more recent than last login (a bot still working after the person left).
- A password is named after a vendor you no longer pay.
- HTTPS is off and Application Passwords are still offered (core requires SSL for a reason; do not “fix” that by disabling the SSL check).

Revoke is the control. Regenerating the login password is not.

## What this inventory is not

This is not a plugin CVE list and not an xmlrpc.php enablement check. It answers one question: after this WordPress user’s login password changed or they left, does any Application Password on that profile still authenticate to `/wp-json/`.

Disclosure: I work on [GuardLMS](https://guardlms.com/en/integrations/wordpress), a product of Ldesign Media. Application Passwords, Basic auth to `/wp-json/`, and per-password revoke are WordPress 5.6+ core as documented in the REST API handbook.

