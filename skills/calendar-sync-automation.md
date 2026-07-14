# calendar-sync-automation

Set up browser-based bidirectional sync between Google Calendar and Microsoft Outlook using ICS export/import, with privacy-safe shared-calendar writes and scheduled automation.

## When to use

Use this skill when a user wants scheduled bidirectional sync between Google Calendar/Gmail and Microsoft Outlook/Work calendar, especially when Google API/OAuth/Cloud Console is unavailable. The pattern is browser automation plus ICS export/import.

## Core approach

1. Do not use Google Calendar API, Google Cloud OAuth, CalDAV, IMAP, or app passwords for calendar operations unless the user explicitly provides a working API path. IMAP is mail-only and does not read/write calendars.
2. Use browser automation at `https://calendar.google.com/` while the user is signed in.
3. For Google -> Outlook: export Google Calendar data from Settings -> Import & export -> Export, parse the downloaded ZIP/ICS files locally, classify future busy events from the primary/personal calendar and shared family calendars, then create/update Outlook events with WorkIQ calendar tools.
4. For Outlook -> Google shared calendar: read Outlook work events with `workiq_list_events`, generate a sanitized ICS file, then use Google Calendar Settings -> Import & export -> Import to import into the target Google calendar selected by name.
5. Use Clawpilot automations for scheduled runs after manually verifying the browser profile is signed in and the Google import/export UI is reachable.

## Current Google Calendar export behavior

1. As of late June/early July 2026, the Gmail/Google Calendar **Import & export -> Export** all-calendars ZIP may only include the primary Gmail calendar ICS, even when the `Family` calendar is visible and editable in the UI. Do not assume the global export contains `Family`.
2. For Google -> Outlook, export the primary calendar from `https://calendar.google.com/calendar/u/0/r/settings/export`, then export `Family` separately from **Settings for my calendars -> Family -> Calendar settings -> Export calendar**.
3. The `Family` calendar export link in the DOM may be a relative `exporticalzip?cexp=...` href. Resolve it against `https://calendar.google.com/calendar/u/0/` to form `https://calendar.google.com/calendar/u/0/exporticalzip?cexp=...`; resolving it against the current settings URL returns an HTML settings page instead of a ZIP.
4. If the Playwright download event times out for the `Family` export, fetch the absolute export URL from the signed-in browser context with credentials included, and require `content-type: application/zip` plus a ZIP containing `Family_...@group.calendar.google.com.ics`. If the response is HTML, the endpoint was resolved incorrectly or Google changed the flow; stop rather than using stale `Family` data.
5. Keep the browser session/profile usable across runs. Prefer the built-in browser automation session; if a fallback Node/Playwright persistent context is used, complete all browser work in one process and release the context so later runs do not hit a locked-profile `Opening in existing browser session` failure.

## Required setup questions

1. Ask for the Google target calendar name for Outlook -> Google imports, for example `Family`.
2. Ask whether Google -> Outlook should copy titles or use generic titles. Default: copy titles if the user wants their work calendar to show personal context; never copy personal descriptions, locations, attendees, or notes unless explicitly requested.
3. Ask what work events should be shared to Google. Default: only events requiring the user to be away from home, such as physical presence, customer days, onsite work, or travel.
4. Ask whether shared Google calendar entries should be sanitized. Default: sanitize work details for shared/family calendars.

## Privacy and safety rules

1. Never expose private work/customer details to shared Google calendars. Use sanitized titles such as `Work/customer day`, `Work onsite`, `Away for work`, or `Work travel` unless the user explicitly approves more detail.
2. For shared/family calendar descriptions, use generic text such as `Copied from work calendar for family visibility.` Do not include customer names, internal meeting names, Teams links, room names, attendees, organizers, notes, or confidential details.
3. For Google personal/shared-family -> Outlook, if copying titles, copy title only; do not copy location, description, attendees, attachments, or notes.
4. Shared/family calendar entries are only for times the user is away from home or must physically be at a customer/office/site. Exclude remote meetings, online events, and meetings the user can do from home.
5. Short away-from-home meetings should be copied as short timed events using their real start/end times, not excluded and not converted to full-day/all-day blocks.
6. Use local ledgers to avoid duplicate imports and persist user-confirmed exclusions. If duplicate behavior or away-from-home status is uncertain, stop and ask rather than importing.
7. Do not send email or Teams notifications as part of the sync unless separately requested and confirmed.
8. If the browser is not signed in, the target Google calendar is missing, or the UI flow changes, stop and report the blocker.

## Recommended local artifacts

Keep these under the user's workspace `Scratchpad` folder:

1. `work-to-family-import-ledger.json`: tracks imported Outlook event IDs, deterministic ICS UIDs, date ranges, hashes, and exclusions for events that should never be re-imported.
2. `google-export-processing-ledger.json`: tracks Google event UIDs copied to Outlook.
3. `work-to-family-import.ics`: generated sanitized ICS for Google import.
4. `google-export-processing-ledger.json`: tracks personal/shared-family Google events copied to Outlook.

Avoid storing secrets. Do not write pasted passwords or tokens to files unless explicitly necessary and safe; this workflow should not need secrets.

## Google -> Outlook implementation outline

1. Navigate to `https://calendar.google.com/calendar/u/0/r/settings/export`.
2. Click Export and wait for the ZIP download in Downloads.
3. Extract ZIP locally and verify whether it contains only the primary Gmail ICS or also shared calendars. If `Family` is missing, do not proceed with only primary data.
4. Export `Family` separately from its calendar settings page. Use the exact `Family` calendar settings URL discovered from the left navigation when possible. If the export href is relative, resolve it to `https://calendar.google.com/calendar/u/0/exporticalzip?cexp=...`, not relative to `/r/settings/calendar/...`.
5. Extract the `Family` ZIP locally and require a `Family` ICS file. Exclude Birthdays/Holidays unless the user asks to sync them.
6. Parse both primary and `Family` ICS files for the next 30 days, expanding RRULE recurrence using installed Python libraries if available: `icalendar` and `recurring_ical_events`. If missing, install with `python -m pip install --user icalendar recurring-ical-events python-dateutil`.
7. Skip any shared-calendar event that is a work-to-shared-calendar artifact: summaries like `Work/customer day`, `Work onsite`, `Away for work`, or `Work travel`, or descriptions containing `Copied from work calendar for family visibility.` This prevents loops back into Outlook.
8. Classify as busy: timed opaque events, or titles indicating appointment, doctor, dentist, travel, school, pickup/dropoff, sports ceremony, graduation, end-of-year event, required personal obligation, or local-language equivalents. Treat birthdays, holidays, reminders, and all-day informational events as free unless clearly blocking.
9. For each busy event, check Outlook for matching marker `[clawpilot-google-sync:<google_uid>]`, `google-export-processing-ledger.json`, and overlapping events. Create idempotent Outlook busy events via `workiq_create_event`. Subject: copied Google title if configured; otherwise `Personal busy`. Body should contain only the sync marker and generic source note. Create events sequentially to avoid Graph mailbox concurrency throttling.
10. Update `google-export-processing-ledger.json` after successful writes.

### Detecting changes to already-synced Google events

A synced Google/personal event can later be moved or renamed. Key the ledger on the event's **stable Google UID** (`googleUid`), NOT on a hash of `uid|start`, so a change is recognized as an update instead of a duplicate.

1. Store `googleUid` (plus `start`, `end`, `title`, and the `outlookEventId`) in each `google-export-processing-ledger.json` record.
2. On each run, match every current Google busy event to prior imports by `googleUid`:
   - **Not in ledger** -> create a new Outlook event (and record its `googleUid`).
   - **In ledger, unchanged** -> skip.
   - **In ledger, but `start`/`end`/`title` differ** -> UPDATE the existing Outlook event (via the stored `outlookEventId`) with `workiq_update_event`, then update the ledger record. Do not create a new event.
3. Keying on `uid|start` alone causes a moved event to look "new" (creating a duplicate at the new time while the old copy goes stale) or, if only the title changed, to be skipped entirely — both are wrong. Always reconcile by `googleUid`.
4. Backfill `googleUid` onto existing future-dated ledger records (match current export by title+start) so changes to previously-synced events are also caught.

## Outlook -> Google shared calendar implementation outline

1. Use `workiq_list_events` to read Outlook events for the next 45 days.
2. Select only candidate work blocks that likely require the user to leave home: explicit travel/onsite/customer-site events, physical customer/office/site locations, long self-owned customer-day blocks, or user-confirmed customer/onsite keywords. Good signals include travel time, location that is clearly not Teams/URL/online, or user-confirmed customer-day keywords.
3. Short meetings are allowed when they require being away from home or physically at a customer/office/site; import them as their actual short timed events. Exclude cancelled events, remote meetings, Teams/online-only meetings, office hours, broad online trainings, optional/community events, and events listed in the ledger's exclusions for non-duration reasons. If a location looks physical but the meeting could be done from home, ask before importing.
4. Generate sanitized ICS with deterministic UID from Outlook event ID. For shared/family target calendars, summary must be generic, for example `Work/customer day`, `Work onsite`, or `Away for work`. Description must be generic. DTSTART/DTEND must match the source event start/end.
5. Compare generated event hashes against `work-to-family-import-ledger.json`. Include only new UIDs or changed events that are not excluded and when update behavior has been validated. If Google import would duplicate changed events, ask the user before importing.
6. Navigate to Google Calendar Import & export page. Select file, choose target calendar by exact name, import.
7. After successful import, update the ledger.

## Away-from-home classification rules (avoid false full-day blocks)

These rules keep short remote days from being copied to the shared calendar as onsite/customer blocks. They were hardened after a real incident: a long Outlook hold titled only with a customer name (e.g. `IAI`) and an **empty location** was wrongly classified as a full `Work/customer day`, and the block persisted on the shared calendar even after the source hold was deleted.

1. **An empty or unknown location is NOT evidence of being on-site.** Never treat a blank location as "not online, therefore physical." Blank location = unknown, which is not a positive signal.
2. **A bare customer/product name in the subject (e.g. `IAI`, `מכבי`) is a weak signal and is NOT sufficient on its own.** It only qualifies when paired with a concrete physical location.
3. Import a block as away-from-home only when at least one of these holds:
   - the subject explicitly states travel / on-site / at-customer (`onsite`, `on-site`, `travel`, `נסיעה`, `אצל הלקוח`, `בלקוח`, `בשטח`, etc.) and it is not an online meeting (short durations allowed); or
   - the location is a concrete physical place (office room codes like `HRZ`/`MPR`/`Conf room`, a venue, a floor/room such as `קומה`/`חדר ישיבות`, or a city/street) AND the block is `busy`/`tentative`; or
   - a customer-name keyword is present AND the location is a concrete physical place.
4. Reserve `Work/customer day` for genuinely long days (roughly 6h+). Shorter physical blocks become `Work onsite` (or `Away for work` for explicit short travel) and MUST preserve the real start/end window — never expand a short meeting into a full day.
5. Ambiguous cases (a long-ish `busy` block with a customer keyword but no physical-location proof) must NOT be auto-imported. Surface them to the user for confirmation instead.
6. Because Google ICS import cannot delete, a wrongly-imported block must be removed from the shared calendar via the browser (open the day view, click the sanitized chip, Delete event), then removed from `work-to-family-import-ledger.json` and added to its `excluded` list so it is never re-imported. Prefer verifying the source Outlook event still exists before relying on a prior import.

## Automation setup

1. Verify Playwright/browser can open Google Calendar and that the user is signed in. If Google asks for passkey/2FA, prompt the user to complete it.
2. Verify the Import & export page is reachable and target calendar appears in the Add to calendar dropdown.
3. Create or update a Clawpilot automation named something like `Bidirectional Calendar Sync` with a schedule such as every weekday at 7am.
4. Keep automation disabled until browser sign-in and import target are verified.
5. Enable only after a dry-run or UI verification succeeds.

## Validation

1. Run one dry pass that lists candidate Google -> Outlook events and Outlook -> Google events without writing.
2. Confirm the target Google calendar dropdown contains the requested shared calendar.
3. Confirm local ledgers exist.
4. After enabling automation, report schedule and scope clearly.
