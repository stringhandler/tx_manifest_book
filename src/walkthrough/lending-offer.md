# Opening the offer

> 📝 **Draft.** This chapter has not been reviewed yet — content may be incomplete or change.

> **Phase 2 of the [lending walkthrough](./lending-protocol.md).** The borrower
> publishes the offer on-chain: `LockCollateral` moves the collateral and all four
> NFTs into covenant UTXOs, advancing the contract from `nfts_issued` to
> `offer_open`.

After [issuing the NFTs](./lending-issuance.md) the borrower holds four tokens and
an instance file, but the collateral is still loose in their wallet. This phase
commits it — the protocol's first move of value into covenant addresses, and the
first use of the `op_return` destination and a pre-build `validations` check.

## `LockCollateral` — publishing the offer

`LockCollateral` takes the collateral and all four NFTs from the borrower's wallet
and moves them into covenant addresses. The collateral goes into the `pre_lock`
UTXO; each NFT goes into a `prelock_script_auth` UTXO (the `script_auth.simf`
wrapper compiled to `PRE_LOCK_COV_HASH`):

```json
"LockCollateral": {
  "inputs": [
    { "id": "collateral_in",   "utxo_source": "wallet", "asset": "compile_params.COLLATERAL_ASSET_ID",
      "amount_sat": { "min_amount": "compile_params.COLLATERAL_AMOUNT" } },
    { "id": "borrower_nft_in", "utxo_source": "wallet", "asset": "compile_params.BORROWER_NFT_ASSET_ID", "amount_sat": 1 },
    { "id": "lender_nft_in",   "utxo_source": "wallet", "asset": "compile_params.LENDER_NFT_ASSET_ID",   "amount_sat": 1 },
    { "id": "first_params_in",  "utxo_source": "wallet", "asset": "compile_params.FIRST_PARAMETERS_NFT_ASSET_ID",  "amount_sat": "compile_params.FIRST_PARAMETERS_ENCODED" },
    { "id": "second_params_in", "utxo_source": "wallet", "asset": "compile_params.SECOND_PARAMETERS_NFT_ASSET_ID", "amount_sat": "compile_params.SECOND_PARAMETERS_ENCODED" },
    { "id": "fee_input", "utxo_source": "wallet", "asset": "lbtc" }
  ],
  "outputs": [
    { "id": "pre_lock_out",       "destination": { "utxo_type": "pre_lock" },            "asset": "compile_params.COLLATERAL_ASSET_ID", "amount_sat": "compile_params.COLLATERAL_AMOUNT" },
    { "id": "borrower_nft_locked","destination": { "utxo_type": "prelock_script_auth" }, "asset": "compile_params.BORROWER_NFT_ASSET_ID", "amount_sat": 1 },
    { "id": "lender_nft_locked",  "destination": { "utxo_type": "prelock_script_auth" }, "asset": "compile_params.LENDER_NFT_ASSET_ID",   "amount_sat": 1 },
    { "id": "first_params_locked",  "destination": { "utxo_type": "prelock_script_auth" }, "asset": "compile_params.FIRST_PARAMETERS_NFT_ASSET_ID",  "amount_sat": "compile_params.FIRST_PARAMETERS_ENCODED" },
    { "id": "second_params_locked", "destination": { "utxo_type": "prelock_script_auth" }, "asset": "compile_params.SECOND_PARAMETERS_NFT_ASSET_ID", "amount_sat": "compile_params.SECOND_PARAMETERS_ENCODED" },
    { "id": "indexer_op_return", "destination": { "type": "op_return" },
      "data": "concat(compile_params.BORROWER_PUB_KEY, compile_params.PRINCIPAL_ASSET_ID)" },
    { "id": "collateral_change", "destination": "change", "asset": "compile_params.COLLATERAL_ASSET_ID", "optional": true },
    { "id": "fee_change",        "destination": "change", "asset": "lbtc", "optional": true }
  ],
  "validations": [
    { "id": "collateral_amount_matches",
      "rule": { "type": "arithmetic", "expr": "collateral_in.amount_sat == compile_params.COLLATERAL_AMOUNT" },
      "error": { "code": "AMOUNT_MISMATCH", "message": "Collateral input amount does not match COLLATERAL_AMOUNT" } }
  ]
}
```

Two things worth pausing on:

- **The `OP_RETURN` advertisement.** `indexer_op_return` writes
  `concat(BORROWER_PUB_KEY, PRINCIPAL_ASSET_ID)` into an unspendable output. It
  carries no value — it's a beacon so an indexer (or a prospective lender's
  wallet) can discover the open offer and the asset it wants by scanning for these
  markers. [Outputs & destinations](../recipes/03-outputs-and-destinations.md)
  introduced the `op_return` destination; here it's used for discovery rather than
  burning.
- **The collateral amount is checked twice.** The `validations` block asserts the
  input matches `COLLATERAL_AMOUNT` *before building* (a fast, friendly error), and
  the `pre_lock` covenant re-checks it *on-chain at spend time*. Validations are a
  convenience; the covenant is the law.

After this broadcasts, the contract is in `offer_open`: collateral and NFTs sit in
covenant UTXOs that only the two `pre_lock` paths can move.

## Run it

`LockCollateral` is run by the **borrower**, using the instance file written at
construction:

```sh
# --- Borrower puts the offer on-chain ---
txw run examples/lending/txmanifest.json LockCollateral \
  --instance examples/lending/lending.instance.json --wallet borrower.json
txw sync --wallet borrower.json
```

With the collateral and NFTs locked behind `pre_lock`, the offer is live. Next, a
lender accepts it — or the borrower withdraws it:
**[Accepting or cancelling the offer](./lending-accept.md)**.
