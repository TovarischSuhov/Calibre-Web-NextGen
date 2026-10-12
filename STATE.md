# STATE: device-scope PR A tests (test writer)

Branch test/device-scope-a from origin/main 65dce676d. Tests only; Sol implements to CONTRACT.md.

## Plan
- [x] server harness probe (tests/unit/device_scope_world.py)
- [x] test_device_scope_koreader_manifest.py: 19 tests, 17 red, 2 golden guards pass on main
- [x] test_device_scope_devices_api.py: 21 tests, 20 red, 1 passes on main (web reader has no control)
- [x] test_device_scope_kobo_unset_guard.py: 4 tests, 2 golden guards pass, 2 red (PATCH 400)
- [x] G6: test_device_scope_foreign_highlights.py: 6 tests, 5 red, 1 passes (unidentified requester);
      foreign_highlight_cleanup_test.lua: 6 cases, 3 red on main, 3 keep-everything guards pass
- [x] frontend/e2e/device-book-scope.spec.ts: typechecks; NOT run (needs container)
- [x] CONTRACT.md, red-on-main run, commit

## Log
- Red run on main: 44 failed / 6 passed across the four Python files (JUnit).
- Seen-red: Kobo guard fails when the Kindle shelf joins kobo_sync; KOReader shelf-only guard likewise.
- Goldens are recorded only from main code: CWNG_RECORD_KOBO_GOLDEN=1 / CWNG_RECORD_KOREADER_GOLDEN=1.
