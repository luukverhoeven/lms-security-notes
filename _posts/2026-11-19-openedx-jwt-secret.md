---
layout: post
title: "LMS and CMS still share JWT_COMMON_SECRET_KEY after the split"
date: 2026-11-19 12:00:00 +0100
permalink: /openedx-jwt-secret/
---

The LMS and CMS hostnames split last year. Learners hit `courses.example.org`. Authors hit `studio.example.org`. Tutor still renders the same `JWT_COMMON_SECRET_KEY` into both.

A hostname split is a DNS and TLS change. JWT audience, issuer, and signing key are a second object. If they still match because they all defaulted to `OPENEDX_SECRET_KEY`, a token minted on one node is acceptable on the other for as long as that secret lives.

Inventory the JWT secret, issuer, and audience after an LMS/CMS split.

## What Tutor actually writes

On Open edX with Tutor, `JWT_COMMON_SECRET_KEY` defaults to `OPENEDX_SECRET_KEY`. Tutor renders it into `JWT_AUTH` for both LMS and CMS: `JWT_SECRET_KEY`, `JWT_AUDIENCE`, `JWT_ISSUER`. `OAUTH_OIDC_ISSUER` is `JWT_COMMON_ISSUER`.

That shared secret is why a session or JWT issued by the LMS can be verified by Studio (CMS) in a default Tutor install. It is convenient. After you put LMS and CMS on different hostnames, it is also leftover coupling: compromise of either host’s config is compromise of the other’s JWT.

`CMS_OAUTH2_KEY_SSO` is a different leftover (the Studio SSO OAuth client). Do not mix it into this check.

Run this only against an Open edX / Tutor instance you own.

## Why the shared secret survives a split

Operators change:

- `LMS_HOST` / `CMS_HOST` in `config.yml`
- TLS certificates
- the public URLs in the MFE configuration

They often do not change:

- `OPENEDX_SECRET_KEY`
- `JWT_COMMON_SECRET_KEY`
- `JWT_AUDIENCE` / `JWT_ISSUER` strings that still embed the old hostname, or that never included a hostname at all

A JWT `iss` / `aud` that still names the old host is a string leftover. A secret that is still the install-time `OPENEDX_SECRET_KEY` is a key leftover. Both belong in the inventory; only the secret is a signing credential.

## The fixture

On a test Tutor 22.x (Verawood) instance you own:

1. Read the rendered LMS and CMS settings (the images write them under `/openedx/config` or via `tutor config printvalue`). Record:

```text
OPENEDX_SECRET_KEY
JWT_COMMON_SECRET_KEY
JWT_SECRET_KEY
JWT_AUDIENCE
JWT_ISSUER
OAUTH_OIDC_ISSUER
LMS_HOST
CMS_HOST
```

2. Diff LMS vs CMS. On a default Tutor install they match on the JWT block. After a deliberate split you should be able to say whether that match is still intended.

3. Decode (do not verify yet) a JWT you obtained as a synthetic user on the LMS. Record `iss`, `aud`, `exp`, `iat`. Compare `iss` / `aud` to `JWT_ISSUER` / `JWT_AUDIENCE`.

```bash
# on a Tutor host you own
tutor config printvalue LMS_HOST
tutor config printvalue CMS_HOST
tutor config printvalue OPENEDX_SECRET_KEY >/dev/null  # exists; do not print it into tickets

# confirm the public LMS still speaks
curl -sS -D - -o /dev/null "$LMS/login" -w "%{http_code}\n"
```

Keep a row:

| Field | LMS | CMS | Match? |
| --- | --- | --- | --- |
| Host | | | should differ after a split |
| `JWT_ISSUER` | | | |
| `JWT_AUDIENCE` | | | |
| `JWT_SECRET_KEY` fingerprint (sha256 prefix) | | | |

Alert when:

- LMS_HOST and CMS_HOST differ, and the JWT secret fingerprint is still the install-time `OPENEDX_SECRET_KEY` with no rotation date.
- `JWT_ISSUER` still names a retired hostname.
- LMS and CMS secrets match and you believed you had issued separate secrets after the split.
- A JWT minted before the split still verifies after you rotated everything except `JWT_COMMON_SECRET_KEY`.

Rotation means new secrets in Tutor config, a restart of LMS and CMS, and a documented invalidation of outstanding JWTs (short `exp` helps; a week-long token does not).

## What this inventory is not

This is not an MFE CORS check and not a leftover Studio SSO client check. It answers one question: after LMS and CMS stopped sharing a hostname, do they still share the JWT signing key and issuer strings Tutor defaulted from `OPENEDX_SECRET_KEY`.

Disclosure: I work on [GuardLMS](https://guardlms.com/en/integrations/open-edx), a product of Ldesign Media. GuardLMS’s Open edX connector is alpha. `JWT_COMMON_SECRET_KEY`, `JWT_AUTH`, and `OAUTH_OIDC_ISSUER` are Tutor-rendered Open edX settings; confirm names on your Tutor 22 / Verawood.2 install before you alert.

