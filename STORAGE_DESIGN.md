# Storage Design

## Summary

The Xelma contract was rebuilt around an **indexed per-user key layout**. The
old design stored every participant's position in a single `Map<Address, T>`
blob under one key, forcing every read/write to deserialise and reserialise
the whole map. The new design stores each user's record under a composite
`(round_id, address)` key — O(1) read/write per user, regardless of round size.

## Key Layout

| Key | Value | Purpose |
|---|---|---|
| `Balance(Address)` | `i128` | per-user balance (unchanged) |
| `PendingWinnings(Address)` | `i128` | per-user pending payout (unchanged) |
| `UserStats(Address)` | `UserStats` | per-user wins/losses (unchanged) |
| `ActiveRound` | `Round` | currently active round metadata |
| `LastRoundId` | `u64` | monotonic round counter |
| **`Position(round_id, user)`** | `UserPosition` | **NEW** — indexed UpDown bet |
| **`PrecisionPosition(round_id, user)`** | `PrecisionPrediction` | **NEW** — indexed precision guess |
| **`RoundParticipants(round_id)`** | `Vec<Address>` | **NEW** — ordered list for resolution iteration |
| `Positions` / `UpDownPositions` / `PrecisionPositions` | _(legacy)_ | kept for read-only migration; never written by new code |

## Operation cost — before vs after

For a round with **N** participants:

| Path | Before (single-map blob) | After (indexed keys) |
|---|---|---|
| `place_bet` (per user) | 1 deserialise + 1 reserialise of N-entry map | 1 has-check + 1 write of single record + 1 append to participant list |
| `place_precision_prediction` | same N-cost as above | same O(1) cost as above |
| `get_user_position` | full N-entry map read | single composite-key read |
| `get_user_precision_prediction` | full N-entry map read | single composite-key read |
| `resolve_round` | 1 read of N-entry map + N stat updates | 1 read of N-entry participant list + N composite-key reads + N stat updates |
| `claim_winnings` | unchanged | unchanged |

The win is at **bet placement**: instead of paying O(N) every time someone
joins the round, the contract now pays O(1). At N = 60 (large-round test),
the old design would deserialise + reserialise a 59-entry map on every new
bet; the new design touches only the single indexed key for that user plus a
small append on the participant list.

## Resolution iteration

Resolution still has to iterate every participant — there is no way around
that, regardless of layout. The participant list (`RoundParticipants(round_id)`)
preserves the iteration order so that the resolution path matches the old
behaviour exactly: same payout formula, same tie-break order, same stats
updates. Per-user position records are then read individually inside the
loop, each as an O(1) ledger entry rather than a slice of one large blob.

## Cleanup

`resolve_round` now performs targeted deletes: it walks the participant list
and removes each `Position(round_id, user)` (and `PrecisionPosition` for
precision mode), then removes the participant list entry itself. The legacy
single-map keys are also `remove`d in case they exist from pre-migration data.

## Determinism guarantees

The refactor preserves every observable output:

- Pool totals (`pool_up` / `pool_down`) are still maintained on the `Round`
  struct exactly as before.
- Refund-on-tie, proportional payout on price move, and precision-mode tie
  splitting all use the same formulas as before.
- `_update_stats_win` / `_update_stats_loss` are called for every participant
  in the same iteration order as before (driven by the participant list, which
  is appended in bet order).

Existing tests (`lifecycle`, `betting`, `resolution`, `mode_tests`, …) all
pass without functional changes. The one test that previously poked at
`DataKey::UpDownPositions` directly (`test_multiple_rounds_lifecycle`) was
updated — it now lets `place_bet` write the indexed key naturally and only
overrides the round pool totals to inject a simulated losing pool.

## Migration notes

- **Legacy keys remain readable.** `get_user_position` falls back to
  `DataKey::Positions` if no indexed entry is present. This lets a
  pre-existing deployment serve historical reads while the next round runs
  against the new layout.
- **Legacy keys are no longer written.** `place_bet` and
  `place_precision_prediction` only emit indexed keys.
- **No data migration required.** Once `resolve_round` is called for any
  in-flight round under the old layout, the contract removes the legacy
  single-map keys and all subsequent rounds use the indexed layout.

## Test coverage

`contracts/src/tests/storage_benchmarks.rs` adds operation-count assertions
for each core path:

- `bench_place_bet_writes_single_user_key` — verifies `place_bet` writes the
  composite-key entry and does **not** write the legacy bulk-map key.
- `bench_place_bet_op_count_assertion` — after 10 bets, exactly 10 indexed
  position keys + 1 participant-list key exist.
- `bench_resolve_cleans_indexed_keys` — after resolution, all per-user keys
  + the participant list are removed.
- `bench_large_round_resolves_correctly` — 60-participant round resolves
  with correct payouts and full storage cleanup.
- `bench_precision_mode_indexed_keys` — same indexed layout used for
  `PrecisionPosition` keys.
