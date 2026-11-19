---
layout: post
title: "The Sakai WebDAV bookmark outlived the account"
date: 2026-11-19 12:00:00 +0100
permalink: /sakai-webdav-bookmark/
---

The Sakai account is disabled. The laptop still has a WebDAV bookmark named “Course Resources” that opens with the old username and a password the password manager never rotated.

Sakai WebDAV is not the portal session. It is a second protocol path onto the same Resources. SSO that covers `/portal` often does not cover it.

Inventory OS-level WebDAV bookmarks and the Sakai accounts that can still authenticate to that path.

## What Sakai WebDAV actually is

In Sakai, a site’s Resources tool has **Transfer Files** / WebDAV. The UI shows a WebDAV URL for that site’s content. Clients are the OS file manager, Cyberduck, WinSCP, or `cadaver`.

Authentication is the Sakai username and password (or the WebDAV-specific credentials your campus configured). It is not the portal JSESSIONID. Community documentation and `sakai.properties` treat WebDAV as its own auth cache. If the campus uses CAS or SAML for the browser, WebDAV still needs a password Sakai will accept — often the native password that SSO made everyone forget, or a separately issued WebDAV secret.

Default post-install credentials (`admin` / `admin`) are a documented change-me. They are a different leftover; do not stop at renaming admin and skip WebDAV.

Run this only against a Sakai you own.

## Why the bookmark outlives the account

A lecturer sets up WebDAV once, on a Mac, so they can drag lecture PDFs into Resources. macOS Keychain stores the password next to the URL. Disabling the Sakai user in the directory, or clicking “Deactivate” in Admin Users, does not open Keychain on that laptop. The bookmark still exists. Whether it still *works* depends on whether Sakai still accepts that username/password on the WebDAV endpoint.

Two leftovers, two checks:

1. **Account leftover** — the Sakai user can still authenticate to WebDAV after you believe they are gone (disable in the IdP did not disable the Sakai-local password; the user is “inactive” in the UI but WebDAV auth cache still has them).
2. **Client leftover** — the OS bookmark and stored password are still on a machine, even if the server now rejects them. That is a credential in a laptop inventory, not an open server door.

A successful WebDAV PROPFIND after disable is the server-side finding. A Keychain entry after disable is the endpoint-offboarding finding.

## The fixture

On a test Sakai 25.x you own:

1. Create user `webdav-canary-001` with a password that exists only in your vault. Add them to a throwaway site as a role that can write Resources.
2. In that site: Resources → Transfer Files. Copy the WebDAV URL. It is typically a path under `/dav/` or `/webdav/` on the same host as the portal; record the exact URL your release shows.
3. From a throwaway local directory, prove write with a client you control:

```bash
# only against a Sakai you own
# cadaver or curl --dav; substitute the URL from Transfer Files
curl -sS -D - -o /tmp/dav-propfind.xml \
  -u 'webdav-canary-001:REDACTED' \
  -X PROPFIND \
  "$SAKAI_DAV_URL"
```

Expect 207 Multi-Status if the user can see the collection. Expect 401 after you disable the account *if* WebDAV consults the same disabled flag as the portal.

4. Disable the user the way your campus actually disables people (Sakai Admin Users deactivate, or IdP disable, or both). Wait the documented cache interval. Repeat the PROPFIND.
5. On one managed test laptop, add the bookmark, then disable the user, then see whether the bookmark still has a stored password.

Keep a row:

| Field | Before disable | After disable |
| --- | --- | --- |
| Portal login | | |
| WebDAV PROPFIND status | | |
| WebDAV auth cache TTL (`sakai.properties`) | | |
| OS bookmark still present | | |

Alert when:

- Portal login fails and WebDAV PROPFIND is still 207 past the auth-cache TTL.
- `admin` / `admin` still authenticates on `/dav` on a host that is not a five-minute install.
- Site membership was removed and WebDAV still lists that site’s collection for the user.
- Offboarding has no step for “remove WebDAV bookmarks on issued laptops”.

Do not invent a second password store inside Sakai unless your release documents one. Many installs authenticate WebDAV with the same username/password as a non-SSO login.

## What this inventory is not

This is not an LTI secret check and not a portal session-cookie flag check. WebDAV is a file-transfer protocol path. The question is: after this test user was disabled the way you disable staff, does the WebDAV URL still accept them, and does a leftover OS bookmark still hold a password.

Disclosure: I work on [GuardLMS](https://guardlms.com/en/integrations/sakai), a product of Ldesign Media. Transfer Files / WebDAV, portal-versus-WebDAV authentication, and the post-install admin password change are Sakai platform behaviour; confirm the exact `/dav` path on your 25.x site before you alert.

