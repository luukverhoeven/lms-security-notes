---
layout: post
title: "The Chamilo installer still owns .env after go-live"
date: 2026-11-19 12:00:00 +0100
permalink: /chamilo-env-install/
---

The Chamilo wizard finished. Students can log in. `main/install/index.php` still answers, `.env` is still owned by the web user, and the admin password you typed into the wizard is the same password that is in the mailbox you used as a placeholder.

Chamilo 3 does not use the old `configuration.php` as the primary config. The wizard writes `.env` (and related files under `config/`). Post-install permissions are a separate step from “the homepage loaded”.

Inventory the installer, the env file, and the admin identity the wizard created.

## What the installer actually leaves behind

Chamilo 3’s web installer is `main/install/index.php`. It asks for the portal URL, database credentials, encryption method, and the first admin user. Encryption method is hard to change later; treat it as a one-way choice on a test site, not something you casually rotate in production.

After a successful install the application reads `.env`. Community install notes tell you to tighten ownership after the wizard, typically by taking `.env` and `config/` away from the web-server user:

```bash
# on a host you own, after the wizard
chown -R root: .env config/
chmod 640 .env
```

The exact group should be the one your php-fpm worker is in if PHP must still read the file. The point is: world-readable `.env` and a web user that can write `.env` are leftover install state.

`main/install/index.php` should stop being a public wizard. On a finished site it should 403, 404, or redirect. A 200 that still renders the installer is the finding, even if a token check inside the wizard refuses to rerun.

Run this only against a Chamilo you own.

## Why it stays open

The person who ran the wizard closed the browser tab when the portal loaded. They did not:

- remove or block `main/install/`
- change ownership of `.env`
- rotate the admin password off the value typed into the wizard (often the same as a shared 1Password item, or `admin` / a campus default)
- confirm the Portal URL in `.env` matches the public hostname (HTTP vs HTTPS, www vs apex)

A reverse proxy can hide `/main/install/` on one vhost and leave it on the canonical host. Check the public hostname students use.

## The fixture

On a throwaway Chamilo 3 you own, after the wizard:

1. Record the Portal URL, admin username, and encryption method the wizard stored.
2. Probe the installer and the env file from the public hostname and from the filesystem.

```bash
HOST=https://chamilo.example.org   # a host you own

curl -sS -D - -o /tmp/chamilo-install.html \
  -w "%{http_code} %{url_effective}\n" \
  "$HOST/main/install/index.php"

# on the box, not through the web server
stat -c '%U %G %a %n' .env config/ || stat -f '%Su %Sg %Mp%Lp %N' .env
```

Expect:

- installer: not a 200 wizard (403/404/30x to login is acceptable; a filled-in install form is not)
- `.env`: not world-readable; not writable by the web user after go-live
- Portal URL inside `.env` equal to the public origin, including scheme

3. Log in as the wizard admin. Change that password to a vaulted value. Confirm the old password fails.
4. Search `.env` for leftover defaults: empty `APP_SECRET`-style keys, `MAILER` pointing at `localhost` with no auth, debug flags, the wizard database password in a second copy under `config/`.

Keep a row:

| Check | Expected after go-live | Observed |
| --- | --- | --- |
| `main/install/index.php` | not a usable wizard | |
| `.env` owner / mode | not writable by www-data; not 666/644 world-readable | |
| Portal URL | matches public origin | |
| Wizard admin password | rotated | |
| Encryption method | recorded, not “change later” | |

Alert when any row is a miss. A 200 on the installer with a “already installed” message is still a miss if it discloses paths or lets you recreate the admin.

## What this inventory is not

This is not a plugin CVE scan and not an uptime check. It answers one question: after this Chamilo 3 left the wizard, is the installer closed, is `.env` locked down, and is the wizard admin no longer using the password that was typed on install day.

Disclosure: I work on [GuardLMS](https://guardlms.com/en/integrations/chamilo), a product of Ldesign Media. Chamilo 3’s `.env` / `config/` layout and `main/install/index.php` are from the platform’s own install path; do not apply `configuration.php` instructions from Chamilo 1.x to a 3.x site.

