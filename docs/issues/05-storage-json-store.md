# Title

Storage foundation: data-dir resolution, atomic JSON store, corruption quarantine, advisory lock

## Summary

Implement `src/storage/paths.ts`, `src/storage/jsonstore.ts`, and `src/storage/lock.ts`: the only code allowed to touch the data directory, providing atomic schema-validated JSON persistence (DESIGN.md §9.1–§9.4).

## Context

Every persistent feature (sessions, replays, profile, custom scenarios) builds on these three modules. `paths.ts` is a security chokepoint (DESIGN.md §18.4): all id→path mapping happens here and nowhere else.

## Scope

- Data-dir resolution: `--data-dir` flag value > `MCPARCADE_DATA_DIR` > `~/.mcparcade`; creation with mode `0700`, subdirs `sessions/`, `replays/`, `scenarios/`.
- `pathFor(kind: "session"|"replay", id)` / `scenarioDir()` — validates via `isValidId(kind, id)` (issue 03) **before** any join, which enforces the kind↔prefix match (`pathFor("session", "rep_…")` is rejected exactly like a malformed id); throws `E_STORAGE` on invalid.
- `readJson<T>(path, {schema, supportedVersion})` / `writeJsonAtomic(path, value)` / `listJsonFiles(dir)`.
- Quarantine: invalid JSON or schema-failing file → rename to `<name>.corrupt-<unixms>`, stderr warn, return `undefined`. **Responsibility split (per DESIGN.md §9.4):** this layer only quarantines and returns `undefined`; mapping a *directly requested* quarantined/missing session id to `E_CORRUPT_SESSION`/`E_SESSION_NOT_FOUND` is the session store's job (issue 06).
- `withLock(name, fn)` advisory lock; `name` must be `"profile"` or a valid session id (`isValidId("session", name)`) — anything else throws `E_STORAGE`. Lock files live directly under the data dir as `<name>.lock`. Callers hold locks only around the read-modify-write itself (§9.3): profile RMW under `withLock("profile")`, session mutations under `withLock(sessionId)` (issue 09 wires the latter).

## Detailed Requirements

1. Atomic write: `writeFile(<path>.tmp.<pid>)` → `fsync` file → `rename` → best-effort `fsync` of directory. File mode `0600`. Serialization: `JSON.stringify(value, null, 0)` + trailing newline.
2. `readJson` validates with the supplied zod schema; `supportedVersion` is passed explicitly by the caller (each store owns its constant). Decision ladder: `stat` size > 8 MiB → corrupt path **without reading the content** (§9.4 hostile-file guard); `schemaVersion` missing or non-integer → corrupt path (quarantine + `undefined`); `schemaVersion > supportedVersion` → throw `E_STORAGE` with an "upgrade mcparcade" message and do **not** quarantine (the file is fine, we're old); otherwise validate with the zod schema (failure → corrupt path).
3. Quarantine never deletes; never throws for unreadable individual files during `listJsonFiles` scans (skip + warn).
4. Lock: create `<name>.lock` with `O_CREAT|O_EXCL` containing `{pid, at}`; on EEXIST read it — if older than 5000 ms, take over (unlink + retry); else retry 3× with 100 ms backoff then throw `E_STORAGE`. Always `unlink` in `finally`.
5. All fs calls via `node:fs/promises`; no sync fs in server paths.
5b. `listJsonFiles(dir)` contract: returns **absolute paths** of regular files whose basename ends in `.json` and does **not** contain `.tmp.` / `.corrupt-` / end in `.lock`, sorted ascending by basename; unreadable entries are skipped with a stderr warning.
6. Export a single `Storage` object constructed with the resolved data dir; nothing else in `src/` may call `path.join` with the data dir (grep-guard test, DESIGN.md §19.4).
7. Windows note: rename-over-existing works on modern Node/NTFS; if it throws `EEXIST`/`EPERM`, retry once after `unlink` (documented; macOS/Linux are the supported platforms in CI, DESIGN.md §20).

## Acceptance Criteria

- [ ] Resolution order test (flag > env > default) with temp dirs.
- [ ] Kill-during-write simulation: write a `.tmp.` file, then a valid target; loader ignores tmp files and reads the target. Atomicity invariant test: a reader loop running concurrently with 200 `writeJsonAtomic` calls to the same path must, on every successful read, obtain JSON that parses and validates against the schema (any `JSON.parse` failure or schema failure = test failure; reads may see the *previous* complete version, never a partial one).
- [ ] Corrupt file (truncated JSON, wrong schema, missing `schemaVersion`) → quarantined with original bytes intact; `readJson` returns `undefined`; a `schemaVersion: 999` file is *not* quarantined and raises `E_STORAGE`.
- [ ] `pathFor` rejects `../evil`, absolute paths, URL-encoded traversal, empty string, wrong prefix, and cross-kind ids (`pathFor("session", "rep_<valid-ulid>")`) — table-driven.
- [ ] `withLock` rejects invalid names (`../x`, uppercase, 40-char junk, `rep_`-prefixed ids) and accepts `"profile"` and a valid `sess_` id.
- [ ] 8 MiB+1 file → quarantined without a full read (spy on `readFile` — only `stat` observed).
- [ ] `listJsonFiles` contract test: mixed dir with `a.json`, `b.tmp.123`, `c.corrupt-1`, `profile.lock`, subdir → returns only `a.json` (absolute), sorted.
- [ ] Lock: two concurrent `withLock` callers serialize (observable via a shared counter file); stale lock (fake old timestamp) is taken over.
- [ ] Grep-guard test passes (`path.join(dataDir` only inside `paths.ts`).

## Validation

- `test/unit/storage/*.test.ts` covering all criteria; runs against `fs.mkdtemp` temp dirs only.

## Dependencies

- 01, 03.

## Non-goals

- Session/replay/profile domain logic (06, 07). Pruning policies (06). Scenario validation (28).

## Design References

- DESIGN.md §9.1–§9.4, §18.4 (T1, T9), §19.4
