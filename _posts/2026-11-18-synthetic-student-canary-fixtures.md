---
layout: post
title: "A synthetic student should never see this course"
date: 2026-11-18 12:00:00 +0100
permalink: /canary-fixtures/
---

A role reset or an LMS upgrade can open a course that was closed last week. Status codes on `/login` will not tell you. A dedicated test course with a marker string will.

Build that course on a test instance you own. Put a unique, harmless token in a page, a file, and a label. Enrol one synthetic student. Leave a second synthetic student unenrolled, with the same role archetype and no extra capabilities. Then assert, on a schedule, that the unenrolled identity cannot read the marker through the browser, the REST/web-service path, and the file URL. Run the same assertions as the enrolled identity so a broken fixture does not look like a successful deny.

Do not put real learner content in this course. Do not reuse a production course. The marker is the only payload.

## What “access” actually means

On Moodle™, `require_login()` lets a user into a course when one of three checks is true: an active enrolment (`is_enrolled()`), the capability `moodle/course:view` (`is_viewing()`), or temporary guest access from an enrolment plugin (`is_guest()`). Guest is a named account. Auto-login and per-course guest enrolment are separate switches. The unenrolled synthetic user must have none of those three paths into the canary course.

The web-service function `core_course_get_contents` first calls `validate_context()` on the course. If the token’s user cannot enter that context, Moodle™ throws `errorcoursecontextnotvalid`. If the user *can* enter the context, the function still walks every section and module. Contents are attached only when `$cm->uservisible` is true. A section that is not user-visible comes back with an empty `modules` array. That is a 200 with a hollow body, not an exception.

On Canvas, a course file carries `visibility_level`: `inherit`, `course`, `institution`, or `public`. The Files API also returns `hidden_for_user` and `locked_for_user`. A wiki page has `published`, `locked_for_user`, and `editing_roles` that can include `public`. `GET /api/v1/courses/:course_id/pages/:url_or_id` returns the page body. `GET /api/v1/courses/:course_id/files` lists files the caller is allowed to see. A file set to `public` is a leak even when enrolment is correct.

## The fixture

Create a course that is unpublished to the catalogue, with a shortname such as `CANARY-AUTHZ-001`. Inside it:

1. A Page (Canvas) or Page/Label (Moodle™) whose body is a single token, for example `CANARY-PAGE-7f3c9e2a`.
2. A file whose contents are a second token, for example `CANARY-FILE-b41d0c88`. On Canvas, set `visibility_level` to `course`. On Moodle™, put it in a File resource with guest access off.
3. Nothing else. No real names, no SIS ids, no production files.

Create two users that exist only for this check. Enrol `canary-student` as a student. Leave `canary-outsider` unenrolled. Issue each a personal web-service token or Canvas access token scoped to that user. Do not use an admin token for the deny path; an admin token will pass `moodle/course:view` and the test will lie.

Record the course id, page URL, file id, and both user ids. That record is the fixture inventory.

## Three probes, two identities

Run these against the test LMS you own. Substitute the ids from the inventory.

Moodle™, as each user:

```bash
# browser path: course home
curl -sS -b "cookies-$USER.txt" -c "cookies-$USER.txt" \
  -o "/tmp/$USER-course.html" -w "%{http_code} %{url_effective}\n" \
  "$HOST/course/view.php?id=$COURSEID"

# web service: modules + file urls
curl -sS "$HOST/webservice/rest/server.php" \
  --data-urlencode "wstoken=$TOKEN" \
  --data-urlencode "wsfunction=core_course_get_contents" \
  --data-urlencode "moodlewsrestformat=json" \
  --data-urlencode "courseid=$COURSEID" \
  -o "/tmp/$USER-contents.json" -w "%{http_code}\n"

# file URL copied from the enrolled student's contents payload
curl -sS -o "/tmp/$USER-file.bin" -w "%{http_code}\n" \
  "$FILEURL&token=$TOKEN"
```

Canvas, as each user:

```bash
AUTH="Authorization: Bearer $TOKEN"

curl -sS -D - -o "/tmp/$USER-course.json" \
  -H "$AUTH" "$HOST/api/v1/courses/$COURSEID"

curl -sS -D - -o "/tmp/$USER-page.json" \
  -H "$AUTH" "$HOST/api/v1/courses/$COURSEID/pages/$PAGEURL"

curl -sS -D - -o "/tmp/$USER-files.json" \
  -H "$AUTH" "$HOST/api/v1/courses/$COURSEID/files"
```

Then search every saved body for the marker tokens. The enrolled student must see both tokens. The unenrolled student must see neither, in HTML, JSON, or the file bytes.

## Why a 403 is not the whole story

A redirect to `/login` is a deny for that browser path. An `errorcoursecontextnotvalid` exception is a deny for the Moodle™ web service. A Canvas `401` or `403` on the page endpoint is a deny for that API call.

A 200 is not a grant until you read the body. Moodle™ will return sections with `uservisible: false` and empty `modules`. Canvas will return a file object with `hidden_for_user: true` or omit the file from the list. A Page whose `body` is missing for this caller is a deny; a Page whose `body` contains the marker is a grant.

Alert on the marker string, not only on the status code. Store the enrolled student’s response hash next to the unenrolled student’s, so a fixture that stopped rendering looks different from a newly opened course.

## After an upgrade

Run the same three probes before a Moodle™ upgrade, after it, and after any role reset. On hosted Canvas, run them after the weekly release window; hosted Canvas has no version number on an admin screen, so date the run from the Canvas release notes you applied.

Compare four things:

- Did the unenrolled identity start seeing a marker?
- Did the enrolled identity stop seeing a marker? That is a broken fixture, not a hardening.
- Did a file `visibility_level` change to `institution` or `public`?
- Did `canary-outsider` pick up `moodle/course:view`, guest enrolment, or a Canvas enrollment it did not have last week?

Keep the course empty of student data. Rotate the marker tokens when someone pastes a screenshot into a ticket. The canary is object-level authorization monitoring. It does not replace a guest-access inventory, an LTI inventory, or a header check. It answers one question: can an identity that should not read this object, read this object, today.

Disclosure: I work on [GuardLMS](https://guardlms.com). The probes above are the platforms’ own APIs and screens.
