# Review resolution record

- Repository: `Saber5656/mcparcade`
- Pull request: #1
- Parent head observed before this addendum: `e5ae993cf27b4ac1faf592cd46c090bc8290688e`
- Scope: existing review threads only; no new Bot review is requested.
- This document records design-level resolutions and focused verification gates. It does not claim implementation or test completion.

## Thread `PRRT_kwDOTN399M6QCw_W`

### Use a per-write temporary filename for atomic writes

- Finding: The existing review thread `PRRT_kwDOTN399M6QCw_W` identifies this contract gap.
- Normative resolution: Generate a unique temporary path per write operation and bind cleanup/rename to that path, so concurrent writes in one process cannot share or delete one another's temporary file.
- Focused verification before resolving this thread: Run concurrent same-target writes and assert each writer has an independent temp path and the final file is one complete valid document.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTN399M6QCw_X`

### Avoid unlinking a lock you no longer own

- Finding: The existing review thread `PRRT_kwDOTN399M6QCw_X` identifies this contract gap.
- Normative resolution: Give each lock acquisition a unique ownership token and remove the lock in cleanup only when the token still matches; stale takeover must invalidate the previous owner's cleanup.
- Focused verification before resolving this thread: Pause an owner across stale takeover and assert its finally block cannot unlink the new owner's lock.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTN399M6QCw_Y`

### Verify every committed Sudoku puzzle for uniqueness

- Finding: The existing review thread `PRRT_kwDOTN399M6QCw_Y` identifies this contract gap.
- Normative resolution: Run the uniqueness/solution consistency validator over every committed puzzle in every difficulty bank, not only the first ten; fail CI on any invalid or duplicate puzzle.
- Focused verification before resolving this thread: Corrupt a puzzle after the sampled range and assert the full bank validation catches it.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTN399M6QCw_Z`

### Create the replay before updating recentReplays

- Finding: The existing review thread `PRRT_kwDOTN399M6QCw_Z` identifies this contract gap.
- Normative resolution: Generate and persist the replay id/content before the profile finalization writes `recentReplays`, then commit the two updates through the defined transaction/recovery path.
- Focused verification before resolving this thread: Finalize a session and assert the profile never references a replay that has not been created and stored.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTN399M6QCw_d`

### Route stale abandonment through finalization

- Finding: The existing review thread `PRRT_kwDOTN399M6QCw_d` identifies this contract gap.
- Normative resolution: Move stale-session abandonment behind the same finalization service used for normal completion so counts, replay creation, rating, and emitted events follow the documented abandonment effects.
- Focused verification before resolving this thread: Advance a session past the stale threshold and assert all finalization side effects occur exactly once.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTN399M6QCw_g`

### Recompute Elo inside the profile lock

- Finding: The existing review thread `PRRT_kwDOTN399M6QCw_g` identifies this contract gap.
- Normative resolution: Acquire the profile lock, re-read the current rating/statistics, compute Elo from that snapshot, and persist the result before releasing the lock; caller-supplied stale ratings are not authoritative.
- Focused verification before resolving this thread: Run concurrent game finalizations and assert rating updates serialize without lost increments.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTN399M6QCw_i`

### Cap replay slices by response bytes

- Finding: The existing review thread `PRRT_kwDOTN399M6QCw_i` identifies this contract gap.
- Normative resolution: Bound replay page construction by serialized response bytes as well as entry count, returning a deterministic cursor/page boundary before the 64 KiB structured-content limit.
- Focused verification before resolving this thread: Use maximum-size actions and assert every page stays within the byte cap and can be continued.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTN399M6QCw_l`

### Define how observe advances the event cursor

- Finding: The existing review thread `PRRT_kwDOTN399M6QCw_l` identifies this contract gap.
- Normative resolution: Make observe take an explicit client cursor/session identifier and return events after that cursor without hidden global mutation; repeated requests with the same cursor are idempotent and the next cursor is explicit.
- Focused verification before resolving this thread: Observe twice from the same cursor and from the returned cursor and assert no duplicate or skipped events.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTN399M6QCw_o`

### Prevent caching of the token-bearing shell URL

- Finding: The existing review thread `PRRT_kwDOTN399M6QCw_o` identifies this contract gap.
- Normative resolution: Avoid token-bearing cache keys by redirecting/scrubbing the URL before the SPA shell, and send `Cache-Control: no-store`, Referrer-Policy, and equivalent headers on the shell and tokenized endpoints.
- Focused verification before resolving this thread: Request `/` with a token through a cache fixture and assert neither the URL nor tokenized response is stored or sent as a referrer.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTN399M6QCw_r`

### Gate final replay rendering on gameVersion

- Finding: The existing review thread `PRRT_kwDOTN399M6QCw_r` identifies this contract gap.
- Normative resolution: Apply the game-version compatibility check to both frame and replay-detail routes; unsupported final state uses the documented unavailable/migration view instead of the current renderer.
- Focused verification before resolving this thread: Open an older-version replay detail and assert it cannot be rendered by an incompatible current renderer.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTN399M6QCw_t`

### Serialize active-session cap checks

- Finding: The existing review thread `PRRT_kwDOTN399M6QCw_t` identifies this contract gap.
- Normative resolution: Perform active-session count, admission, and persistence under one cross-process lock/transaction so concurrent starts cannot both pass the cap.
- Focused verification before resolving this thread: Start two processes at the cap boundary and assert at most the configured number becomes active.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTN399M6QCw_v`

### Key HTTP throttling by source before session

- Finding: The existing review thread `PRRT_kwDOTN399M6QCw_v` identifies this contract gap.
- Normative resolution: Apply rate limiting by trusted source IP (with explicit proxy trust handling) before client-controlled `mcp-session-id`, then optionally add a session bucket.
- Focused verification before resolving this thread: Rotate session ids from one source and assert the source budget is shared rather than reset per session.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Thread `PRRT_kwDOTN399M6QCw_w`

### Allow seers to report dead targets

- Finding: The existing review thread `PRRT_kwDOTN399M6QCw_w` identifies this contract gap.
- Normative resolution: Permit a seer result for a target that died during the same night when the state-machine evidence shows the target was valid at action time; reject only unknown/invalid targets.
- Focused verification before resolving this thread: Run a same-night death plus seer-result fixture and assert the result is accepted and replayed without allowing arbitrary dead-target claims.
- Resolution record status: addressed in this design addendum; implementation/test completion is not claimed here.

## Bot review policy

The existing Bot review is not re-triggered for this PR. Replies and thread resolution are performed only after the focused verification conditions above are recorded.