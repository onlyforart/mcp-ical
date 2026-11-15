# Review of SPEC-mcp-ical.md

Review date: 2026-11-15. Checked against the code at upstream commit `6746f87` and the live upstream PR diffs (Omar-V2/mcp-ical). No code changes were made as part of this review.

## Blocker

**1. The project only runs on macOS.** The review was done on a Linux machine. PyObjC/EventKit won't install there, so Phase 0's baseline run, every integration test and Phases 2, 10 and 11 can't run on it. Even the mocked unit tests fail, because `models.py` and `ical.py` import `EventKit` at the top. Options:

- run the plan on a Mac;
- or add a step that moves the EventKit imports into functions, so the unit tests (timezone parsing, auth messages, flat parameters) can also run on Linux or in CI, while integration tests stay Mac-only.

## The PR descriptions are out of date

**2. PR #18 is no longer just "delete".** It now has 14 commits and includes:

- the closed PR #19's five timezone "layers" and the token-saving compact output from closed PR #28;
- updates to a single occurrence (`update_event` gains `occurrence_date` and `update_future_events`);
- **attendees added via AppleScript**: it runs `osascript`, starts Calendar.app, and sends real invitations;
- a committed `.mcp.json` with a hardcoded `/Users/tamm/...` path;
- a broken `.gitignore` line (`.DS_Store.mcp.json`);
- a `list_events` that **rejects naive datetimes**, which contradicts the Phase 4 policy (naive means local time).

"Merge PR #18" in Phase 7 would bring all of this in, and rewrite `create_event`/`update_event` after Phases 4 and 6 have already changed them. That means heavy conflicts and undoing earlier work. **Suggestion:** cherry-pick only the delete and occurrence-matching code, and list each of the other features as take or drop.

**3. PR #18's naive-datetime handling is "local first, then UTC", not "UTC" as the spec says.** The fix is still to remove the UTC fallback. Otherwise a naive time could match, and delete, the wrong occurrence.

**4. PR #26 does not flatten the recurrence fields. It removes `recurrence_rule` from `create_event` entirely.** Merging it as is would drop recurring-event creation. Phase 6 has to add the flat recurrence parameters itself and name them, e.g. `recurrence_frequency`, `recurrence_interval`, `recurrence_end_date`, `recurrence_count`, `recurrence_days_of_week`. Also, #26 parses the datetimes outside its `try`, so bad input raises an exception instead of returning an error string.

**5. PR #32 isn't covered by #18.** It adds a second tool, `delete_events_in_range`. As written that tool is dangerous: for each occurrence of a recurring event it calls `delete_event`, which deletes this and all future occurrences. Record an explicit "drop", or a redesign.

**6. Phase 5 has existing code.** The closed PR #24 is exactly the `NSURL` fix for #23. Cherry-pick it and add the tests.

## Wrong or incomplete fixes

**7. Phase 4 doesn't fix the real cause of #17 and #20.** `Event` is a plain `@dataclass`, so the `FlexibleDateTime`/`convert_datetime` validator never runs. `start_time` and `end_time` stay as raw `NSDate`, which prints in UTC (`2025-12-20 11:00:00 +0000`). Changing `convert_datetime` alone won't affect event output. The conversion has to happen explicitly in `Event.from_ekevent`, or `Event` has to become a pydantic model.

**8. Phase 3 and Phase 4 conflict.** PR #20 returns naive local datetimes (`tzinfo=None`) for all-day events, and Phase 4 requires every datetime to carry an offset. Either do Phase 4 first, or say up front that Phase 4 will revise #20's output.

Separately, #20 sets the all-day end to 23:59:59. Check whether EventKit's all-day `endDate` is the same day or the next midnight, because multi-day events depend on it.

**9. "Run under two time zones" only works for pure-Python tests.** `time.tzset()` doesn't reset Foundation's cached `NSTimeZone.systemTimeZone`, and EventKit uses that. Integration tests (Phase 7 "under at least two timezones") need separate `TZ=... pytest` processes, plus a call to `NSTimeZone.resetSystemTimeZone()`. Calendar.app's own time zone override setting can also interfere.

**10. Phase 5's "invalid URL returns an error" may be hard to trigger.** On recent macOS, `NSURL.URLWithString_` percent-encodes most input instead of returning `None`. Define what counts as invalid, e.g. requiring a scheme.

## Gaps in the spec

**11. No way to clear a field.** Phase 6 says unset fields are left unchanged, and Phase 5 wants the URL clearable on update. Pick one convention, e.g. an empty string or empty list means clear. It applies to location, notes, url, alarms and recurrence.

**12. Updates to recurring events have no scope.** Today `update_event` always saves with `EKSpanFutureEvents`, which, given the master ID, changes the whole series. Phase 7 gives delete a this/future/all scope, but update gets none. Also consider replacing the confusing `delete_entire_series=True` + `occurrence_date` (which means "this and future") with `scope: "this" | "future" | "all"` on both tools. The fork isn't bound to #18's API.

**13. Datetime parameters not all named.** Phase 4 says every datetime parameter becomes `str`. Name them all, including `occurrence_date` on delete/update and the recurrence end date.

**14. The existing tests break the test-calendar rule.** The session cleanup deletes **any** calendar whose title starts with `test_calendar_`, and calendars are created in the iCloud source by default. Moving to `mcp-ical-test-` should be a Phase 0 step, not something left implicit. Two related bugs in `ical.py`:

- `_delete_calendar`'s "verify deletion" compares an ID against a list of names, so it always passes;
- the error path in `_create_calendar` reuses a stale `source` variable.

**15. Test imports will collide after the upgrade.** The existing tests import `src.mcp_ical...` while PR #20's tests import `mcp_ical...`, so the modules load twice under different names. Exception and class identity checks then fail, and the logger is set up twice. `[tool.pytest]` in `pyproject.toml` is only read by pytest 9+; the lock has 8.3.4, so it starts taking effect after `uv lock --upgrade`. Standardise on `mcp_ical` imports in Phase 1.

**16. Phase 1 doesn't set an upper version bound.** The lock has `mcp 1.2.1`. Add `mcp<2` (or decide to target 2.x), so `mcp.server.fastmcp` can't break later.

**17. The current alarm logic disagrees between create and update.** `update_event` subtracts 1440 minutes for all-day events, but only when `all_day` is in the request; `create_event` doesn't. Add a test for it in Phase 6.

**18. Phase 0 should match the existing checkout.** The repo was already cloned with `origin` pointing at upstream. As of this review, `origin` now points at the fork (`git@github.com:onlyforart/mcp-ical.git`) and upstream is the `upstream` remote, so Phase 0 step 1 is done.

**19. Some issues are missing from the scope.** #3, #14 and #29 (permission prompt / launching) belong under Phase 2/10, #5 (time zone) under Phase 4, and #6 (Cursor `create_event` failure) is fixed by Phase 6. List them so the issue mapping is complete.

**20. The smoke test needs a few clarifications.** There's no create/delete-calendar tool, so creating and deleting `mcp-ical-smoke` is done by hand in Calendar.app. In step 5, "the 3rd occurrence" should mean the originally 3rd one. `docs/install.md` also needs updating (its Claude Desktop JSON example is missing commas), not just `README.md`. In Phase 10, use the absolute path from `uv tool install`, in case `~/.local/bin` isn't on Claude Code's `PATH`.

## Open decisions

- Which Mac runs this, and should the unit tests also work on Linux (point 1)?
- From PR #18, keep or drop: AppleScript attendees, compact `list_events` output and deduplication of all-day events, and occurrence-level updates?
- The "clear a field" convention, and whether to switch to `scope` (points 11 and 12)?
- PR #32's bulk delete: drop, or redesign?
