---
layout: post
title: "Unpublishing the Sakai site left the LTI 1.1 secret"
date: 2026-11-19 16:00:00 +0100
permalink: /sakai-lti11-secret/
---

The Sakai site is unpublished. The LTI 1.1 tool is still there with the same consumer key and shared secret the vendor mailed in 2019.

Unpublishing a site hides it from the site tab. It does not rotate LTI 1.1 credentials. LTI 1.1 launches authenticate with that shared secret. Anyone who still has the secret can sign a launch if the endpoint still accepts it.

Inventory LTI 1.1 secrets on sites that are no longer taught, and on tools whose vendors you no longer pay.

## What an LTI 1.1 secret actually is

Sakai’s External Tools / LTI tool configuration stores a consumer key and a shared secret per tool (site-level or admin-level, depending on how it was created). LTI 1.1 (IMS) signs the launch POST with OAuth 1.0 HMAC-SHA1 using that secret. The LMS posts to the tool’s launch URL with `lis_person_*` fields according to the privacy flags on the tool.

LTI 1.3 / Advantage uses a different key type (asymmetric, platform issuer, JWKS). Do not mix them. A leftover 1.3 developer key is a Canvas-shaped problem; this note is 1.1 shared secrets in Sakai.

Sakai 25.x still supports LTI 1.1 tools because campuses still have them. Unpublished sites keep their tool registrations unless someone deletes the tool or the site.

Run this only against a Sakai you own.

## Why the secret survives

End of term: unpublish the site, archive the roster, keep the site “in case we run it next year”. The External Tool row stays. The vendor still has the secret they were sent in 2019. The launch URL may still be live on the vendor side.

A site delete that does not cascade tool secrets into a vault rotation leaves the vendor holding a credential that will work again the moment someone republishes or copies the site.

Admin-level (pluggable, available in all sites) tools are worse: unpublishing one course site does not touch them at all.

## The fixture

On a test Sakai 25.x you own:

1. Create a throwaway site. Add an LTI 1.1 external tool you control (a test launch endpoint you own, not a vendor). Set a consumer key `canary-lti11` and a secret from the vault.
2. Launch once as a synthetic instructor. Confirm your endpoint receives a signed POST.
3. Unpublish the site. Do not delete the tool.
4. Read the tool configuration (Site → External Tools, or the admin LTI tool list). Confirm the secret is still the same value (Sakai may show bullets; use the same admin view you would use to rotate it).
5. From a signed test POST your endpoint already knows how to verify, confirm the secret still verifies. If the unpublished site still allows an instructor to open the tool, that is the leftover launch path. If it does not, the leftover is the secret itself, sitting in the database and in the vendor’s mail.

Keep a row per LTI 1.1 tool:

| Site id / published | Tool title | Launch URL | Key | Secret last rotated | Vendor still contracted |
| --- | --- | --- | --- | --- | --- |
| | | | | | |

Alert when:

- A site is unpublished (or the term ended) and the LTI 1.1 secret is older than that term.
- The launch URL host is a vendor you no longer have a DPA with.
- An admin-level 1.1 tool is enabled in “all sites” and nobody can name an owner.
- Site copy duplicated the secret into a new site you did not inventory.

Rotation means new secret in Sakai *and* at the tool. Changing it only in Sakai breaks the launch; that is a successful revoke if the vendor should no longer launch.

## What this inventory is not

This is not WebDAV, not a 1.3 JWKS inventory, and not a Substack-style “LTI 1.1 leftover secrets” Moodle™ note. Sakai stores these on the site / admin External Tool object. The question is: after this site was unpublished, does the LTI 1.1 shared secret still exist unchanged.

Disclosure: I work on [GuardLMS](https://guardlms.com/en/integrations/sakai), a product of Ldesign Media. LTI 1.1 consumer key/secret, site unpublish, and External Tools are Sakai platform behaviour on 25.x; confirm the admin labels on your site.
