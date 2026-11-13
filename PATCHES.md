# vendored rquickjs-sys patch set

Base: `rquickjs-sys 0.12.2` → `quickjs-ng 0.15.1`.

This copy is vendored so Breeze can carry quickjs-ng security fixes and a
subset of the Telegram quickjs performance work without waiting for a new
`rquickjs-sys` release. Rust bindings are unchanged; only `quickjs/quickjs.c`
and `quickjs/dtoa.c` are patched.

## Security backports

| Issue | Upstream commit | Change |
|---|---|---|
| `for (await using ...)` UaF | Telegram `ca3594dae5` | Reject `await using` in a for head outside async functions; suspending opcodes throw `InternalError` in a `JS_FUNC_NORMAL` frame |
| FinalizationRegistry unregister UAF | `7ca4ac52` | Unlink matching entries first, free in a second pass |
| Map.prototype.clear UAF | `3ad65ada` | Pin the current record, read `next` after deletion |
| reset_weak_ref abort | `45c5b9b7` | Detach the WeakRef before the second pass |
| direct eval OOB | `3e236cb0` | Reject `closure_var_count > JS_MAX_LOCAL_VARS` |
| mapped arguments OOB | `8f0d3b61` | Reject `var_ref_count > JS_MAX_LOCAL_VARS` |
| misc parser crashes | `b02310a1` | Check `new_label()` failures |
| parser scope OOM | `9184c729` | Propagate `push_scope()` failures |
| negative scope_level | `a11af653` | Defense-in-depth checks in `add_scope_var`/`pop_scope` |

The `await using` fix is hand-adapted to 0.15.1; the others are applied from
the Telegram `tdesktop-fixes` branch. Regression tests live in
`rust/rquickjs_playground/tests/quickjs_security.rs`.

## Performance backports

Applied from the Telegram quickjs performance branch:

- `94cd057efe` — `QJS_DISABLE_DUMPS` (drop dump-hook checks without NDEBUG)
- `74f2249f40` — Grisu3 shortest base-10 float printing (`dtoa.c`)
- `e69387d23f` — let/const assignment codegen (`put_loc_check` directly)
- `aa443fe24c` — `str.length` inline
- `f9bf3e3950` — string literals without `JS_AtomToValue`
- `ddcd8e58c7` — JSON.stringify skips the `toJSON` lookup when impossible
- `6b6c1f2832` — RegExp bytecode cache (16 patterns per context)
- `7ead61ab67` — `indexOf`/`includes` memchr/memcmp, string Map hash cache
- `4b387c6e02` — inline number comparisons and typed-array stores
- `2adec3fef4` — C function calls skip the `JS_CallInternal` frame
- `e142acb847` — cheaper frame setup and direct C getter calls
- `b989db6196` — `put_field` adds a plain-object property directly
- `7de45b949d` — array method lookup fast path (`get_length`/`get_field`/`get_field2`)

Manual helpers required by the above:

- `js_arith_to_float64` — copied from quickjs-ng 0.17.0 (used by `4b387c6e02`)
- `js_exotic_skip_for_key` — copied from `7de45b949d` (used by the JSON
  `toJSON` fast path and the three interpreter lookup sites)

## Not applied

These depend on 0.16/0.17 code or on each other and did not apply cleanly to
0.15.1:

- `286a5617f1` JSON.stringify direct output
- `9383ba4207` JSON.stringify recursion frame
- `27787d3cdb` per-runtime arena slabs
- `e65afb876b` shape cache
- `70044443db` keep one empty arena per size class
- `ee1ef122ca` inline small property arrays
- `cc381516db` arena page size atomics
- `7d1085a1c6` UBSan slab tag shift
- `7bff43c0c6` Map/Set hash avalanche — hand-ported, but
  `tests::fetch::fetch_abort_concurrent_requests_do_not_leave_pending_http_tasks`
  hangs with it on 0.15.1; dropped until the interaction is understood

## Updating

1. Re-vendor `rquickjs-sys` from crates.io.
2. Re-apply the patches above (Telegram `tdesktop-fixes` branch / quickjs-ng
   commits).
3. Run `cargo test --lib -- --test-threads=1` and
   `cargo test --test quickjs_security` in `rust/rquickjs_playground`.
