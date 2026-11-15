# SPEC: mcp-ical local fork (up-to-date install for Claude Code)

## Goal

Produce a maintained personal fork of [Omar-V2/mcp-ical](https://github.com/Omar-V2/mcp-ical) that:

- merges the useful open PRs,
- fixes known open bugs that have no PR,
- applies one consistent timezone policy,
- installs as a stable command and is registered with Claude Code at user scope.

Upstream appears unmaintained (no merges for a long time; the last maintainer activity found is a review on PR #1). Treat the fork as the source of truth. Do not open PRs upstream unless asked.

## Context

- Python 3.12+, `uv`, PyObjC (EventKit), FastMCP over stdio. Entry point `mcp_ical.server:main`.
- Main files: `src/mcp_ical/server.py` (MCP tools), `src/mcp_ical/ical.py` (`CalendarManager`, EventKit calls), `src/mcp_ical/models.py` (pydantic models, datetime conversion).
- Current tools: `list_calendars`, `list_events`, `create_event`, `update_event`, plus resource `calendars://list`.
- iCloud calendars are reached through the local Calendar.app/EventKit store. No CalDAV work is in scope.
- macOS only.

## Ground rules

- Work test-first. For every behaviour change, write or adapt a failing test before the implementation.
- One phase per commit (or small series of commits). Keep the suite green at the end of every phase, apart from the known pre-existing failure recorded in Phase 0.
- When merging a PR, review its diff first. Do not take it blindly. Note in the commit message what was taken, what was changed, and why.
- Integration tests create and delete temporary calendars and events in the real Calendar store, which syncs to iCloud. Never point tests at existing user calendars. All test calendars must be prefixed (e.g. `mcp-ical-test-`) and cleaned up in teardown, including on failure.
- Stop and ask before anything destructive outside test calendars.

---

## Phase 0: Fork and baseline

1. Fork upstream to my GitHub account, clone locally, create branch `main` tracking the fork, add `upstream` remote.
2. `uv sync`, run the full test suite.
3. Record the baseline in `NOTES.md`: pass/fail counts and the names of any failing tests. (PR #20 reports 16 passing, 1 pre-existing failure. Confirm.)
4. Fetch all open upstream PR branches locally for review: `git fetch upstream pull/<n>/head:pr-<n>` for #1, #13, #18, #20, #26, #27, #32.

**Done when:** baseline recorded; PR branches available locally.

## Phase 1: Dependencies

1. `uv lock --upgrade`. Commit `uv.lock`.
2. Tighten lower bounds in `pyproject.toml` to the resolved versions of `mcp[cli]`, `pyobjc`, `loguru`.
3. Fix any FastMCP API breakage from the upgrade.
4. Add `pytest-asyncio` to dev deps if any test or merged PR needs it (PR #1 review asked for it).

**Done when:** server starts (`uv run mcp-ical` waits on stdio without error); suite matches baseline.

## Phase 2: Permission handling audit

Inspect how `CalendarManager` requests access in `ical.py`.

- On macOS 14+ it must use `requestFullAccessToEventsWithCompletion_`; fall back to `requestAccessToEntityType_completion_` only on older systems.
- Check `EKEventStore.authorizationStatusForEntityType_` and handle `FullAccess`, `WriteOnly`, `Denied`, `Restricted`, `NotDetermined` with clear error strings. A `WriteOnly` grant must produce an explicit message, since reads will silently return nothing.
- Error text should tell the user that the grant belongs to the app that launched the process (their terminal app) and point to System Settings > Privacy & Security > Calendars.

**Done when:** unit tests (mocked status) cover each authorization state's message.

## Phase 3: All-day events on wrong day (merge PR #20)

Review and merge `pr-20`. All-day events must be converted to local time and normalised to 00:00:00 start / 23:59:59 end on the correct local date. Keep its timezone-agnostic tests.

**Done when:** PR tests pass; suite at baseline or better.

## Phase 4: Timezone policy (fixes issues #17, #25)

Define and enforce one policy across all tools:

- **Input:** any ISO 8601 datetime, with or without offset, including a trailing `Z`. A naive datetime means the system local timezone. Not UTC.
- **Internal:** convert to timezone-aware datetimes before building `NSDate`.
- **Output:** every datetime returned to the client includes the UTC offset and zone abbreviation (e.g. `2026-11-16 14:00 GMT (+00:00)`).

Implementation notes:

- Add one parsing helper (e.g. `parse_client_datetime(str) -> datetime`) used by every tool that accepts datetimes. Tool parameters that currently take `datetime` should take `str` and call the helper, so behaviour does not depend on client-side or FastMCP coercion (issue #25).
- Replace naive `datetime.fromtimestamp()` in `models.py` with aware conversion (`datetime.fromtimestamp(ts, tz=timezone.utc).astimezone()`), and add `%Z`/offset to `Event.__str__` formats (issue #17).
- Make sure Phase 3's all-day logic still holds under the new policy.
- Fix the `list_events` docstring: it says `end_date` is optional but the parameter is required. Decide one way (suggest: optional, defaulting to end of `start_date`'s day) and make code and docstring agree.

Tests:

- naive, `Z`, `+00:00`, `-06:00`, `+13:00` inputs all resolve to the correct instant;
- output strings include offset;
- run key tests under at least two `TZ` values (e.g. `Europe/London`, `Pacific/Auckland`) via monkeypatching or `time.tzset()`.

**Done when:** all above tests pass.

## Phase 5: URL crash (fixes issue #23)

`ekevent.setURL_(...)` in `create_event` and `existing_ek_event.setURL_(...)` in `update_event` pass a Python `str`; EventKit expects `NSURL`, and the server crashes with `NSInvalidArgumentException`.

- Convert with `Foundation.NSURL.URLWithString_(url)`.
- If the result is `None` (invalid URL), return a clear error string; do not crash.
- Allow clearing the URL on update.

**Done when:** integration test creates and updates an event with a URL and reads it back; invalid URL returns an error.

## Phase 6: Flat tool parameters (merge PR #26, extend to update)

Review and merge `pr-26` (`create_event` takes flat top-level params: required `title`, `start_time`, `end_time`; optional `calendar_name`, `location`, `notes`, `alarms_minutes_offsets`, `url`, `all_day`, recurrence fields).

Apply the same treatment to `update_event`: `event_id` plus flat optional fields. Unset fields must not be changed. Build the pydantic request objects internally so `CalendarManager` is unchanged.

Datetime params use the Phase 4 helper.

**Done when:** tests call both tools with flat params; partial updates leave other fields untouched.

## Phase 7: Delete (merge PR #18, adjusted)

Use PR #18 as the delete implementation. It supersedes #1, #27 and (after diff review) probably #32; record the decision for each in `NOTES.md`.

Required behaviour of `delete_event`:

- `event_id` only: delete a non-recurring event.
- `event_id` + `occurrence_date`: delete only that occurrence.
- `event_id` + `occurrence_date` + `delete_entire_series=True`: delete that occurrence and all future ones.
- `event_id` + `delete_entire_series=True`: delete the whole series.
- Not found: clear "not found" message (cf. #27's `NoSuchEventException` handling).
- After deletion, verify the event or occurrence is actually gone and report failure if not (this was the silent-failure bug in the PR #1 review).

**Required change to PR #18:** it interprets naive datetimes as UTC. Change it to the Phase 4 policy (naive = local). Occurrence matching must compare instants, not wall-clock strings.

Tests: keep #18's four integration tests; add one where the occurrence is given with a non-UTC offset, and one where local time is not UTC.

**Done when:** all delete tests pass under at least two timezones.

## Phase 8 (optional): Busy/free (issue #21)

Add optional `availability` (`busy` | `free` | `tentative` | `unavailable`) to create and update, mapping to `EKEventAvailability`. Include availability in event output.

## Phase 9 (optional): Calendars with the same name (issue #16)

- `list_calendars` returns, per calendar: title, account/source title, calendar identifier, and whether it is the default.
- Every tool that accepts `calendar_name` also accepts `calendar_id`. If a name matches more than one calendar, return an error listing the candidates (account + id) instead of guessing.
- Event output includes the calendar's account so results are not ambiguous.

Do **not** merge PR #13 (Calendar Groups).

## Phase 10: Install and register with Claude Code

1. `uv tool install --editable .` from the fork so `mcp-ical` is on `PATH`. (Avoids the `ModuleNotFoundError: No module named 'mcp_ical'` in issue #15.)
2. `claude mcp add --scope user ical -- mcp-ical`
3. Launch Claude Code from a terminal, call `list_calendars`, and accept the macOS Calendars prompt. Note in `README` that the permission belongs to the launching terminal app, so each terminal (Terminal, iTerm, Ghostty, VS Code) asks once.
4. Update `README.md`: Claude Code install steps, tool list, timezone policy, delete semantics.

## Phase 11: Smoke test (manual, in a scratch calendar)

Create a calendar `mcp-ical-smoke`, then through Claude Code:

1. Create an all-day event tomorrow. Check it lands on the right day in Calendar.app.
2. Create a timed event with a URL and a reminder. Check time, URL, alarm.
3. Create a weekly recurring event (4 occurrences).
4. Delete the 2nd occurrence only; check 3 remain.
5. Delete from the 3rd occurrence onward; check 1 remains.
6. Update the remaining event's time using an ISO string with an explicit offset; check the result.
7. Delete the calendar.

Record results in `NOTES.md`.

---

## Out of scope

- PR #13 Calendar Groups, PR #12 badge.
- Issue #22 travel time (EventKit is not known to expose it publicly; verify before attempting).
- Issue #11 current-time tool (Claude Code already has the date).
- Issues #10 (Dockerfile), #31 (Gemini).
- CalDAV / non-macOS support.

## Deliverables

- Fork with phases 0 to 7 and 10 complete; 8 and 9 if time allows.
- `NOTES.md`: baseline, per-PR decisions, smoke test results.
- Updated `README.md`.
- Green test suite (apart from any documented pre-existing failure, ideally fixed).
