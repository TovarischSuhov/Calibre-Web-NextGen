# CONTRACT: per-device book scope, PR A (G1 + G6)

The tests on this branch pin these seams. Specs: `state/kindle-kids/specs/2026-10-12-per-device-scope.md` and
`2026-10-12-sync-removal.md` (PR A = model + UI + KOReader, G1 + G6). Everything not written here is the implementer's
choice: the storage shape (columns or table), the migration, and the module layout.

## 1. Devices API (`cps/annotations.py`)

`PATCH /api/annotations/devices/<public_id>` accepts
`{"book_scope": {"mode": M, "shelf_ids": [int], "magic_shelf_ids": [int]}}`, alone or with the existing `label` and
`share_place` fields.

- `M` is one of `"unset"`, `"library"`, `"my_library"` or `"shelves"`. The id lists matter only for `shelves`; they are
  optional for the other modes.
- Success answers 200 with the device JSON (the same row shape as the list), including the new `book_scope`.
- The answer is 400 `{"error": "invalid_book_scope"}`, with nothing changed, for any of these:
  - a mode that isn't one of the four;
  - a `book_scope` that isn't an object;
  - `shelves` with no shelf and no magic shelf;
  - an id that isn't an int, doesn't exist, or isn't the user's own (another user's public shelf included);
  - `my_library` while My Library is off for the user.
- A web-reader device answers 400 (any error code).
- Another user's device answers 404.

`GET /api/annotations/devices` rows for `kobo` and `koreader` devices carry this `book_scope`:

```
{"mode": "unset"|"library"|"my_library"|"shelves",
 "shelf_ids": [int], "magic_shelf_ids": [int],          # as saved
 "account_mode": "library"|"my_library"|"shelves",      # what "unset" means for this account today
 "effective_mode": ...,                                 # what the device gets now (account_mode when unset or fallen back)
 "fallback": null | {"reason": "shelf_deleted"|"shelf_unavailable"},
 "choices": {"shelves": [{"id", "name"}], "magic_shelves": [{"id", "name"}], "my_library": bool}}
```

- `account_mode` is `shelves` for a shelf-only account, `my_library` when My Library is on and the account isn't
  shelf-only, and `library` otherwise.
- `choices` lists only the user's own shelves.
- Web-reader rows have `book_scope` null or absent. A row without `book_scope` (an older server) shows no control.
- New devices start at `unset`.
- Saving one device leaves the others, and that device's `label` and `share_place`, unchanged.

## 2. Resolver (`cps/services/ereader_scope.py`)

`membership(user, *, device=None, ...)` keeps today's keyword arguments.

- `device=None`, or a device whose mode is `unset` or which has fallen back, returns exactly `membership(user)`.
- `shelves` returns `Membership(mode="shelves", book_ids=frozenset(raw ids on the chosen shelves), reliable=True)`.
  The ids are taken before visibility; `held_book_ids` still intersects them with the visible set. The shelf's
  `kobo_sync` flag is ignored.
- `library` returns `book_ids=None` when My Library is off.
- `my_library` returns the user's My Library ids.

## 3. KOReader manifest (`GET /kosync/syncs/library`)

The device is the one named by `X-CWNG-Device-ID` / `X-CWNG-Device-Name`, as `kosync_place.request_device` resolves it.

- Its books are `held_book_ids` for that device's membership. Visibility still removes books that are restricted,
  hidden, archived or outside My Library.
- In `shelves` mode, `scope == "shelves"`, and `scope_shelves` lists the chosen shelves as
  `{"id": shelf.uuid, "name", "kind": "shelf"}`. The `scope` string for `my_library` isn't pinned.
- **An unset device is byte-for-byte main** (golden recorded on main 65dce676d:
  `tests/unit/fixtures/device_scope_koreader_unset_golden.json`). Book checksums are verified against the file and
  masked, because the test EPUBs are zipped at the current time.
- `revision` changes whenever the device's scope changes, including a switch between two shelves that hold the same
  books. Another device on the account keeps its revision, so `if_revision` still answers `{"unchanged": true}` for it.
- A cursor walk that began before a scope change finishes on its original revision. The new scope arrives on the next
  sync.
- **Magic shelves:** a magic shelf picked for a device is evaluated through
  `magic_shelf.get_book_ids_for_magic_shelf(shelf_id, ...)` (the tests stub it, with the global
  `config_kobo_sync_magic_shelves` on). If that raises, the answer is 503 `{"error": "scope_unavailable"}`, never a
  short list (#468).
- **Fallback:** if every chosen shelf is deleted, or now belongs to another user, the device serves the account scope
  with 200. It is never empty and never an error.
  - The row reports `fallback.reason` as `shelf_deleted` or `shelf_unavailable`, with `effective_mode == account_mode`.
  - Deletion goes through the real `POST /api/v1/shelves/<id>/delete`; the ownership change is a direct `user_id`
    update.
- A scope change writes no rows in `device_reading_position`, `book_read_link`, `annotations` or `kobo_reading_state`.

## 4. Kobo (unchanged in PR A)

An unset Kobo's sync response is byte-for-byte main, for a shelf-only account and for a whole-library account. That
holds for a fresh device and for one holding everything (golden `tests/unit/fixtures/device_scope_kobo_unset_golden.json`;
the sync token's random `delivery_epoch` is made deterministic in the test). It stays true while a sibling KOReader on
the account has a device scope.

## 5. G6, server: the annotation pull (`GET /kosync/syncs/annotations/<digest>`)

For a request identified as a registered KOReader device, the body gains `foreign_annotation_ids: [str]`.

- The list holds the `annotation_id` of every row of this user and book whose `origin_device_id` is set and isn't this
  device, hidden rows included. Kobo, web-reader and other-KOReader origins all count as foreign.
- It excludes rows whose origin was never recorded (NULL), other users' rows, and other books' rows.
- It is `[]` when nothing is foreign.
- An unidentified requester gets the key absent or empty.

## 6. G6, plugin (`cwngsync.koplugin`)

During `CWNGSync:syncAnnotations`, after a successful pull, a local highlight is removed from the device only when it
carries `cwng_id` and that id is in `foreign_annotation_ids`.

- These are kept:
  - highlights made here (no `cwng_id`), even if the local id equals a listed one;
  - received highlights that aren't listed;
  - everything, when the pull fails or the key is absent, empty or not a list.
- A removed foreign id is never pushed back and never named in the push's `deleted` list, in this sync or any later
  one. Removing it must not leave it in the watermark as "known to both sides".
- Where the logic lives is the implementer's choice: `koreader_annotations_provider.lua`, `sync_logic.lua`, a new
  required module, or `CWNGSync:` methods in `main.lua`. The Lua harness loads `syncAnnotations` plus every
  `CWNGSync:` method it reaches through `self:` calls. A new `local function` in `main.lua` won't be loaded.

## 7. UI (Devices page, `/app/account/devices`)

- Each row whose `book_scope` is present has a group whose accessible name includes "Books on this device". It holds:
  - radios "Same as my account setting", "Everything in my library", "Only these shelves", and a My Library radio
    only when `choices.my_library` is true;
  - shelf checkboxes named by the shelf name;
  - a "Save" button.
- Choosing shelves with none ticked sends nothing.
- The help text and the two fallback notices use the msgids in `tests/unit/test_device_scope_devices_api.py`
  (`CONTROL_STRINGS`). The fr and nl catalogs translate every one of them.
- Rows without `book_scope` get no group, so existing `share-place.spec.ts` checkbox counts hold.

## Ambiguities and how the tests resolve them

- **Partial shelf loss** (one of two chosen shelves deleted) isn't tested; the implementer chooses and documents it.
- **Magic shelf with the global `config_kobo_sync_magic_shelves` off:** not tested.
- **A Kobo row's control:** the API stores and returns a Kobo scope, as the spec gives Kobo the control. Kobo sync
  ignores it until PR B, and nothing on this branch tests a scoped Kobo's sync. Whether to hide or disable the Kobo
  control until PR B is the implementer's or reviewer's call.
- **Cursor walks across a scope change** finish on the old revision (consistent with today's `(user, revision)`
  cursor cache).
