# Settling: repay, liquidate, withdraw

> 📝 **Draft.** This chapter has not been reviewed yet — content may be incomplete or change.

> **Phase 4 of the [lending walkthrough](./lending-protocol.md).** The loan is
> active. It ends one of two ways — the borrower repays and reclaims the
> collateral, or the lender liquidates it after the deadline — and the lender
> finally withdraws their principal plus interest from a vault.

By now the collateral sits behind the `lending` covenant, the borrower holds the
principal, and the lender holds the Lender NFT. This phase resolves the loan. It
reuses the two-path covenant shape from
[accepting the offer](./lending-accept.md), adds an
**absolute timelock**, and introduces the last piece — the
[`asset_auth.simf`](https://github.com/stringhandler/txmanifest-wallet/blob/main/examples/lending/asset_auth.simf)
**principal vault**.

## A detour: `ClaimLoanFunds`

`SetupLending` delivered the principal to the borrower's `p2pk` address — a
covenant output, not a plain wallet UTXO. `ClaimLoanFunds` sweeps it into the
wallet so the borrower can actually use it. It's the
[Hello World receive](../recipes/01b-hello-world-receive.md) spend, unchanged: one
`p2pk` input, a Schnorr signature, one wallet output.

```json
"ClaimLoanFunds": {
  "inputs": [
    { "id": "principal_in", "utxo_source": { "utxo_type": "p2pk" },
      "witnesses": {
        "SPEND_PATH": { "type": "taproot_leaf", "source": { "type": "formula", "expr": "p2pk_leaf" } },
        "SIGNATURE":  { "type": "Signature", "sig_type": "sig_hash_all",
                        "source": { "type": "wallet", "key": "$params.BORROWER_PUB_KEY" } } } },
    { "id": "fee_input", "utxo_source": "wallet", "asset": "lbtc", "optional": true }
  ],
  "outputs": [
    { "id": "principal_to_borrower", "destination": "wallet", "asset": "compile_params.PRINCIPAL_ASSET_ID", "amount_sat": "principal_in.amount_sat" },
    { "id": "fee_change", "destination": "change", "asset": "lbtc", "optional": true }
  ]
}
```

This is **not a lifecycle transition** — the loan is still `loan_active` whether or
not the borrower has swept the principal. It's housekeeping, included to show that
a covenant payout is just another UTXO you spend normally once it's yours.

**Run it** as the borrower. `ClaimLoanFunds` also needs `--state` so the tool knows
*which* `p2pk` UTXO to claim — the live one the state file recorded when
`SetupLending` paid the principal out:

```sh
txw run examples/lending/txmanifest.json ClaimLoanFunds \
  --instance examples/lending/lending.instance.json \
  --state examples/lending/lending.state.json \
  --wallet borrower.json
txw sync --wallet borrower.json
```

## The `lending` covenant: repay or liquidate

[`lending.simf`](https://github.com/stringhandler/txmanifest-wallet/blob/main/examples/lending/lending.simf)
is the offer covenant's sibling — same `Either<(), ()>` `PATH` selector, different
two outcomes:

```rust
fn main() {
    match witness::PATH {
        Left(params: ())  => { loan_repayment_path(); },   // borrower repays
        Right(params: ()) => { loan_liquidation_path(); }, // lender seizes after expiry
    }
}
```

`PATH::LEFT` is repayment — anyone can fund it, but it only succeeds if it routes
principal + interest to the lender's vault. `PATH::RIGHT` is liquidation — gated by
a timelock instead of a signature.

## `RepayLoan` — borrower settles (`PATH::LEFT`)

The borrower returns principal **plus interest** and gets the collateral back. The
covenant computes the interest on-chain (`calculate_interest`, the same basis-point
math the instance stored as `PRINCIPAL_INTEREST_AMOUNT`) and demands the repayment
land in the lender's vault:

```rust
fn loan_repayment_path() {
    assert!(jet::eq_32(jet::current_index(), 0));
    ensure_input_and_output_assets_with_amount_eq(0, 0, param::COLLATERAL_ASSET_ID, param::COLLATERAL_AMOUNT); // in0→out0 collateral back
    let first  = ensure_input_and_output_assets_eq(1, 2, param::FIRST_PARAMETERS_NFT_ASSET_ID);
    let second = ensure_input_and_output_assets_eq(2, 3, param::SECOND_PARAMETERS_NFT_ASSET_ID);
    ensure_input_and_output_assets_with_amount_eq(3, 4, param::BORROWER_NFT_ASSET_ID, 1);
    // …unpack & validate terms…
    let owed: u64 = calculate_principal_with_interest(principal_amount, interest_rate);
    ensure_asset_with_amount(1, false, param::PRINCIPAL_ASSET_ID, owed); // out1 = principal+interest
    ensure_script_hash(1, false, param::LENDER_PRINCIPAL_COV_HASH);      // …into the vault
    ensure_output_is_op_return(2);  // burn the params + borrower NFT
    ensure_output_is_op_return(3);
    ensure_output_is_op_return(4);
}
```

Same off-by-one as `SetupLending`: the repayment is injected at output 1, shifting
the NFT outputs down. The collateral returns to the borrower's wallet (output 0),
the Parameter and Borrower NFTs are **burned** (the loan is over), and the
principal + interest goes to the vault — *not* directly to the lender. The manifest
mirrors that layout:

```json
"RepayLoan": {
  "inputs": [
    { "id": "lending_in", "utxo_source": { "utxo_type": "lending_collateral" },
      "witnesses": { "PATH": { "type": "simplicityhl", "value": "Left(())" },
                     "SPEND_PATH": { "type": "taproot_leaf", "source": { "type": "formula", "expr": "lending_leaf" } } } },
    { "id": "first_params_in",  "utxo_source": { "utxo_type": "lending_script_auth" }, "…": "…" },
    { "id": "second_params_in", "utxo_source": { "utxo_type": "lending_script_auth" }, "…": "…" },
    { "id": "borrower_nft_in",  "utxo_source": { "utxo_type": "lending_script_auth" }, "…": "…" },
    { "id": "repayment_in", "utxo_source": "wallet", "asset": "compile_params.PRINCIPAL_ASSET_ID",
      "amount_sat": { "min_amount": "compile_params.PRINCIPAL_AMOUNT + compile_params.PRINCIPAL_INTEREST_AMOUNT" } },
    { "id": "fee_input", "utxo_source": "wallet", "asset": "lbtc", "optional": true }
  ],
  "outputs": [
    { "id": "collateral_returned",        "destination": "wallet", "asset": "compile_params.COLLATERAL_ASSET_ID", "amount_sat": "compile_params.COLLATERAL_AMOUNT" },
    { "id": "principal_interest_to_vault","destination": { "utxo_type": "lender_principal_vault" }, "asset": "compile_params.PRINCIPAL_ASSET_ID",
      "amount_sat": "compile_params.PRINCIPAL_AMOUNT + compile_params.PRINCIPAL_INTEREST_AMOUNT" },
    { "id": "first_params_burned",  "destination": { "type": "op_return" }, "asset": "compile_params.FIRST_PARAMETERS_NFT_ASSET_ID",  "amount_sat": "first_params_in.amount_sat" },
    { "id": "second_params_burned", "destination": { "type": "op_return" }, "asset": "compile_params.SECOND_PARAMETERS_NFT_ASSET_ID", "amount_sat": "second_params_in.amount_sat" },
    { "id": "borrower_nft_burned",  "destination": { "type": "op_return" }, "asset": "compile_params.BORROWER_NFT_ASSET_ID", "amount_sat": 1 },
    { "id": "repayment_change", "destination": "change", "asset": "compile_params.PRINCIPAL_ASSET_ID", "optional": true },
    { "id": "fee_change",       "destination": "change", "asset": "lbtc", "optional": true }
  ]
}
```

The repayment input requires `PRINCIPAL_AMOUNT + PRINCIPAL_INTEREST_AMOUNT` — the
derived interest the [constructor](./lending-issuance.md#one-derived-field-the-interest-amount)
computed and stored. The Lender NFT isn't touched here; it's still in the lender's
wallet, waiting to unlock the vault. `RepayLoan` is the lifecycle's **cooperative**
happy path: state moves to `repaid`.

## `LiquidateAfterExpiry` — lender seizes (`PATH::RIGHT`)

If the borrower never repays, the lender takes the collateral — but only after the
deadline. The covenant enforces that with an **absolute timelock**:

```rust
fn loan_liquidation_path() {
    assert!(jet::eq_32(jet::current_index(), 0));
    ensure_input_and_output_assets_with_amount_eq(0, 0, param::COLLATERAL_ASSET_ID, param::COLLATERAL_AMOUNT);
    let first  = ensure_input_and_output_assets_eq(1, 1, param::FIRST_PARAMETERS_NFT_ASSET_ID);   // identity mapping
    let second = ensure_input_and_output_assets_eq(2, 2, param::SECOND_PARAMETERS_NFT_ASSET_ID);
    ensure_input_and_output_assets_with_amount_eq(3, 3, param::LENDER_NFT_ASSET_ID, 1);
    // …unpack & validate terms…
    jet::check_lock_height(loan_expiration_time);  // ← block height must be ≥ expiry
    ensure_output_is_op_return(1);
    ensure_output_is_op_return(2);
    ensure_output_is_op_return(3);
}
```

`check_lock_height` is the on-chain half of an `nLockTime`/CLTV timelock: the spend
is only valid once the chain reaches `LOAN_EXPIRATION_TIME`. Note the mapping is the
**identity** here (no payout injected) and the gating token is the **Lender NFT**,
which the lender brings from their wallet — there's no signature, holding the NFT
*is* the authorisation. The collateral lands in the lender's wallet; the NFTs burn.
The manifest also declares a friendly pre-build `validations` check so you get a
clean error instead of a rejected broadcast if you try too early:

```json
"validations": [
  { "id": "expiry_reached",
    "rule": { "type": "arithmetic", "expr": "current_block_height >= compile_params.LOAN_EXPIRATION_TIME" },
    "error": { "code": "TIMELOCK_NOT_ELAPSED", "message": "Loan has not yet expired. Cannot liquidate before LOAN_EXPIRATION_TIME." } }
]
```

This is the **unilateral** escape hatch for the lender, the mirror image of the
borrower's `CancelOffer`: state moves to `liquidated`. Compare the *relative*
timelock (`check_lock_distance`) in the [Last Will](../recipes/04b-last-will.md) —
that one counts blocks since the UTXO was created; this one names an absolute
height.

## The principal vault: `asset_auth.simf`

Whichever way the loan settled honestly, `RepayLoan` parked the lender's money in a
`lender_principal_vault` UTXO rather than paying the lender directly. Why the extra
hop? Because at repayment time the transaction is driven by the *borrower* — they
shouldn't dictate the lender's receiving address, and the lender may be offline. The
vault holds the funds under a covenant only the **Lender NFT holder** can open:

```rust
fn auth_with_burn_check(input_asset_index: u32, output_asset_index: u32) {
    ensure_asset_and_amount_eq(input_asset_index,  true,  param::ASSET_ID, param::ASSET_AMOUNT);  // LENDER_NFT, 1
    ensure_asset_and_amount_eq(output_asset_index, false, param::ASSET_ID, param::ASSET_AMOUNT);
    match param::WITH_ASSET_BURN {
        true  => ensure_output_is_op_return(output_asset_index),  // and the NFT must be burned
        false => {},
    }
}
```

Compiled with `ASSET_ID = LENDER_NFT_ASSET_ID`, `ASSET_AMOUNT = 1`, and
`WITH_ASSET_BURN = true`, it says: *to move what's in this vault, you must spend the
Lender NFT as an input and burn it as an output.* The NFT is a one-shot key.

## `ClaimPrincipalWithInterest` — lender withdraws

The lender opens the vault by co-spending and burning their NFT, sending the
principal + interest wherever they like:

```json
"ClaimPrincipalWithInterest": {
  "params": { "lender_destination": { "type": "address" } },
  "inputs": [
    { "id": "vault_in", "utxo_source": { "utxo_type": "lender_principal_vault" },
      "witnesses": {
        "INPUT_ASSET_INDEX":  { "type": "formula", "expr": "index_of(lender_nft_in)" },
        "OUTPUT_ASSET_INDEX": { "type": "formula", "expr": "index_of(lender_nft_burned)" },
        "SPEND_PATH": { "type": "taproot_leaf", "source": { "type": "formula", "expr": "lender_principal_vault_leaf" } } } },
    { "id": "lender_nft_in", "utxo_source": "wallet", "asset": "compile_params.LENDER_NFT_ASSET_ID", "amount_sat": 1 },
    { "id": "fee_input", "utxo_source": "wallet", "asset": "lbtc" }
  ],
  "outputs": [
    { "id": "principal_interest_out", "destination": "params.lender_destination", "asset": "compile_params.PRINCIPAL_ASSET_ID", "amount_sat": "vault_in.amount_sat" },
    { "id": "lender_nft_burned", "destination": { "type": "op_return" }, "asset": "compile_params.LENDER_NFT_ASSET_ID", "amount_sat": 1 },
    { "id": "fee_change", "destination": "change", "asset": "lbtc", "optional": true }
  ]
}
```

The `index_of(...)` formula is the key trick: the covenant needs to know *which*
input is the NFT and *which* output burns it, but those positions depend on how the
builder ordered the transaction. Rather than hard-code indices, the witnesses are
computed with [`index_of(id)`](../recipes/07-formulas-and-derived-params.md), which
resolves an input/output `id` to its final position — so the covenant's
`INPUT_ASSET_INDEX` / `OUTPUT_ASSET_INDEX` always point at the right slots. State
moves to `settled`, and the loan is fully wound down.

> **Why a vault instead of paying the lender directly — and "accumulation."** The
> `asset_auth` pattern decouples *when funds are paid in* from *when they're
> collected*. A lender running many loans accrues a vault per loan, each unlocked by
> its own Lender NFT, and can sweep them on their own schedule from any wallet that
> holds the NFTs — without the borrowers needing the lender's addresses. It's a
> reusable building block: an asset-gated, burn-on-spend output.

## Run it

The two settlements are run by different parties. The **borrower** repays —
returning principal + interest and reclaiming the collateral (you'll have swept the
principal with [`ClaimLoanFunds`](#a-detour-claimloanfunds) above):

```sh
# --- Borrower repays ---
txw run examples/lending/txmanifest.json RepayLoan \
  --instance examples/lending/lending.instance.json --wallet borrower.json
```

Then the **lender** drains the vault:

```sh
# --- Lender collects principal + interest ---
txw run examples/lending/txmanifest.json ClaimPrincipalWithInterest \
  --instance examples/lending/lending.instance.json --wallet lender.json
```

Or, if the borrower defaulted, the **lender** liquidates once the chain has passed
`LOAN_EXPIRATION_TIME` (before then, the `TIMELOCK_NOT_ELAPSED` validation stops
you):

```sh
txw run examples/lending/txmanifest.json LiquidateAfterExpiry \
  --instance examples/lending/lending.instance.json --wallet lender.json
```

Remember to `sync` each wallet after a broadcast.

> **Liquidation needs the chain at the expiry height.** `check_lock_height` is an
> absolute-height lock, so on testnet you either set a near-future
> `LOAN_EXPIRATION_TIME` at construction or wait for the height to arrive. Treat
> the liquidation path as illustrative until the chain catches up to the deadline
> you chose.

## You've reached the end

That's the whole protocol: a borrower and a lender transacting a collateralised
loan with no escrow, every rule — terms, amounts, timelock, payout routing —
enforced by five small Simplicity covenants and four NFTs. Along the way you've now
seen, working together, every concept the cookbook introduced one recipe at a time:
[covenant UTXO types](../recipes/05-covenant-utxo-types.md),
[multiple spending paths](../recipes/06-multiple-spending-paths.md),
[issuance & NFTs](../recipes/08-issuance-and-nfts.md),
[formulas & derived params](../recipes/07-formulas-and-derived-params.md),
[hooks & tapleaf compute](../recipes/09-hooks-and-tapleaf.md), and the
[class / instance model](../recipes/10-instance-state-constructors.md).

For the precise rules behind anything here, the authoritative reference is
[`Spec.md`](https://github.com/stringhandler/tx_manifest_spec/blob/main/Spec.md).
