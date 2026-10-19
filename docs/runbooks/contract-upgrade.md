# Contract upgrade and rollback runbook

This runbook covers a Soroban WASM upgrade for the MarketPay contract. Keep
the incident channel open, announce the maintenance window, and record the
operator, network, contract ID, old version, new version, and transaction
hashes before starting.

## Preflight

```bash
export NETWORK=testnet                 # use mainnet only during an approved window
export IDENTITY=marketpay-deployer
export CONTRACT_ID=C...                 # deployed contract ID
export VERSION=1.4.0
rustup target add wasm32v1-none
cargo build --manifest-path contracts/marketpay-contract/Cargo.toml \
  --target wasm32v1-none --release
sha256sum contracts/marketpay-contract/target/wasm32v1-none/release/marketpay_contract.wasm
stellar contract id asset --asset native --network "$NETWORK" >/dev/null
```

Compare the resulting SHA-256 and the intended version with the reviewed
release artifact. Never upgrade from an unreviewed local build.

## Upgrade

Install the reviewed WASM and capture its hash:

```bash
WASM=contracts/marketpay-contract/target/wasm32v1-none/release/marketpay_contract.wasm
WASM_HASH=$(stellar contract install --wasm "$WASM" --source "$IDENTITY" \
  --network "$NETWORK")
stellar contract invoke --id "$CONTRACT_ID" --source "$IDENTITY" \
  --network "$NETWORK" -- upgrade \
  --new_wasm_hash "$WASM_HASH" --new_version "$VERSION"
stellar contract invoke --id "$CONTRACT_ID" --source "$IDENTITY" \
  --network "$NETWORK" -- get_version
```

Verify `get_version`, `get_upgrade_count`, and representative read-only escrow
queries. Soroban upgrades preserve contract storage; do not run migrations
unless the release specifically includes one.

## Rollback

Rollback when post-upgrade reads fail, transactions revert unexpectedly, or a
security incident is confirmed. Pause user-facing writes, announce the
rollback, and reinstall the last known-good WASM hash:

```bash
stellar contract install --wasm /secure/releases/marketpay-<previous>.wasm \
  --source "$IDENTITY" --network "$NETWORK"
stellar contract invoke --id "$CONTRACT_ID" --source "$IDENTITY" \
  --network "$NETWORK" -- upgrade \
  --new_wasm_hash "$PREVIOUS_WASM_HASH" --new_version "$PREVIOUS_VERSION"
```

Record the rollback transaction, restore writes only after smoke checks pass,
and open a follow-up incident review. Testnet rehearsal is required before a
mainnet upgrade; this repository does not claim that a live testnet rehearsal
was performed by this documentation change.
