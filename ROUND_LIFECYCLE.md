# Round Lifecycle

## States

```
                    ┌─────────────┐
                    │   Settled   │ ◄─────────────────────────────────┐
                    └─────────────┘                                    │
                           │                                           │
              create_round (admin)                          resolve_round (oracle)
                           │                                           │
                           ▼                                           │
                    ┌─────────────┐                                    │
                    │   Active    │ ───────────────────────────────────┘
                    └─────────────┘
                    Bet window open
                    (ledger < bet_end_ledger)
                           │
                    bet window closes
                    (ledger ≥ bet_end_ledger)
                           │
                    Run window closes
                    (ledger ≥ end_ledger)
```

## Single-Active-Round Invariant

**At most one round may be in the Active state at any point in time.**

This is enforced by `assert_no_active_round`, a guard helper called at the
start of `create_round`, before any storage writes:

```rust
fn assert_no_active_round(env: &Env) -> Result<(), ContractError> {
    if env.storage().persistent().has(&DataKey::ActiveRound) {
        return Err(ContractError::RoundAlreadyActive);
    }
    Ok(())
}
```

If an active round is detected the function returns `ContractError::RoundAlreadyActive`
immediately. No storage keys are mutated — the round counter (`LastRoundId`)
and the existing `ActiveRound` entry both remain unchanged.

## Entrypoints That Enforce the Guard

| Entrypoint | Guard applied |
|---|---|
| `create_round` | `assert_no_active_round` before any write |

Any future entrypoint that could create a round must also call
`assert_no_active_round` before touching storage.

## Error Mapping

| Rust variant | Code | TypeScript message |
|---|---|---|
| `ContractError::RoundAlreadyActive` | 20 | `"RoundAlreadyActive"` |

## Storage Keys Affected by the Guard

| Key | Written on success | Written on failure |
|---|---|---|
| `DataKey::ActiveRound` | ✅ New round struct | ❌ Unchanged |
| `DataKey::LastRoundId` | ✅ Incremented | ❌ Unchanged |
| `DataKey::UpDownPositions` | ✅ Cleared | ❌ Unchanged |
| `DataKey::PrecisionPositions` | ✅ Cleared | ❌ Unchanged |

## Round Resolution

The oracle calls `resolve_round` after `end_ledger` is reached. On success:
- `ActiveRound` is removed — the invariant is reset and a new round can be created.
- Participant positions (`UpDownPositions`, `PrecisionPositions`) are removed.
- Pending winnings for winners are written to `PendingWinnings(address)`.

## Claiming Winnings

Users call `claim_winnings` any time after a round resolves. The pending amount
is added to their balance and the `PendingWinnings` entry is removed atomically.
