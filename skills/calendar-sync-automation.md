# calendar-sync-automation

Set up browser-based bidirectional sync between Google Calendar and Microsoft Outlook using ICS export/import, with privacy-safe shared-calendar writes and scheduled automation.

## When to use

Use this skill when a user wants scheduled bidirectional sync between Google Calendar/Gmail and Microsoft Outlook/Work calendar, especially when Google API/OAuth/Cloud Console is unavailable. The pattern is browser automation plus ICS export/import.

## Core approach

1. Do not use Google Calendar API, Google Cloud OAuth, CalDAV, IMAP, or app passwords for calendar operations unless the user explicitly provides a working API path. IMAP is mail-only and does not read/write calendars.
2. Use browser automation at `https://calendar.google.com/` while the user is signed in.
3. For Google -> Outlook: export Google Calendar data from Settings -> Import & export -> Export, parse the downloaded ZIP/ICS files locally, classify future busy events, then create/update Outlook events with WorkIQ calendar tools.
4. For Outlook -> Google shared calendar: read Outlook work events with `workiq_list_events`, generate a sanitized ICS file, then use Google Calendar Settings -> Import & export -> Import to import into the target Google calendar selected by name.
5. Use Clawpilot automations for scheduled runs after manually verifying the browser profile is signed in and the Google import/export UI is reachable.

## Required setup questions

1. Ask for the Google target calendar name for Outlook -> Google imports, for example `Family`.
2. Ask whether Google -> Outlook should copy titles or use generic titles. Default: copy titles if the user wants their work calendar to show personal context; never copy personal descriptions, locations, attendees, or notes unless explicitly requested.
3. Ask what work events should be shared to Google. Default: only events requiring the user to be away from home, such as physical presence, customer days, onsite work, or travel.
4. Ask whether shared Google calendar entries should be sanitized. Default: sanitize work details for shared/family calendars.

## Privacy and safety rules

1. Never expose private work/customer details to shared Google calendars. Use sanitized titles such as `Work/customer day`, `Work onsite`, `Away for work`, or `Work travel` unless the user explicitly approves more detail.
2. For shared/family calendar descriptions, use generic text such as `Copied from work calendar for family visibility.` Do not include customer names, internal meeting names, Teams links, room names, attendees, organizers, notes, or confidential details.
3. For Google personal -> Outlook, if copying titles, copy title only; do not copy location, description, attendees, attachments, or notes.
4. Shared/family calendar entries are only for times the user is away from home or must physically be at a customer/office/site. Exclude remote meetings, online events, and meetings the user can do from home.
5. Use local ledgers to avoid duplicate imports and persist user-confirmed exclusions. If duplicate behavior or away-from-home status is uncertain, stop and ask rather than importing.
6. Do not send email or Teams notifications as part of the sync unless separately requested and confirmed.
7. If the browser is not signed in, the target Google calendar is missing, or the UI flow changes, stop and report the blocker.

## Recommended local artifacts

Keep these under the user's workspace `Scratchpad` folder:

1. `work-to-family-import-ledger.json`: tracks imported Outlook event IDs, deterministic ICS UIDs, date ranges, hashes, and exclusions for events that should never be re-imported.
2. `google-export-processing-ledger.json`: tracks Google event UIDs copied to Outlook.
3. `work-to-family-import.ics`: generated sanitized ICS for Google import.

Avoid storing secrets. Do not write pasted passwords or tokens to files unless explicitly necessary and safe; this workflow should not need secrets.

## Google -> Outlook implementation outline

1. Navigate to `https://calendar.google.com/calendar/u/0/r/settings/export`.
2. Click Export and wait for the ZIP download in Downloads.
3. Extract ZIP locally. Identify the user's primary calendar ICS; prefer the calendar with the Gmail address or primary calendar name, and exclude Birthdays/Holidays/Family unless the user asks to sync them.
4. Parse ICS for the next 30 days, expanding RRULE recurrence using installed Python libraries if available: `icalendar` and `recurring_ical_events`. If missing, install with `python -m pip install --user icalendar recurring-ical-events python-dateutil`.
5. Classify as busy: timed opaque events, or titles indicating appointment, doctor, dentist, travel, school, pickup/dropoff, required personal obligation, or local-language equivalents. Treat birthdays, holidays, reminders, and all-day informational events as free unless clearly blocking.
6. For each busy event, check Outlook for matching marker `[clawpilot-google-sync:<google_uid>]` and overlapping events. Create idempotent Outlook busy events via `workiq_create_event`. Subject: copied Google title if configured; otherwise `Personal busy`. Body should contain only the sync marker and generic source note.
7. Update `google-export-processing-ledger.json` after successful writes.

## Outlook -> Google shared calendar implementation outline

1. Use `workiq_list_events` to read Outlook events for the next 45 days.
2. Select only candidate work blocks that likely require the user to leave home: long self-owned customer-day blocks, explicit travel/onsite/customer-site events, or physical customer/office/site locations. Good signals include all/most-day blocks, travel time, location that is clearly not Teams/URL/online, or user-confirmed customer-day keywords.
3. Exclude cancelled events, remote meetings, Teams/online-only meetings, office hours, broad online trainings, recurring short meetings, optional/community events, and events listed in the ledger's exclusions. If a location looks physical but the meeting could be done from home, ask before importing.
4. Generate sanitized ICS with deterministic UID from Outlook event ID. For shared/family target calendars, summary must be generic, for example `Work/customer day`, `Work onsite`, or `Away for work`. Description must be generic.
5. Compare generated event hashes against `work-to-family-import-ledger.json`. Include only new UIDs or changed events that are not excluded and when update behavior has been validated. If Google import would duplicate changed events, ask the user before importing.
6. Navigate to Google Calendar Import & export page. Select file, choose target calendar by exact name, import.
7. After successful import, update the ledger.

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
