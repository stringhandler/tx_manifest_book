# Accepting or cancelling the offer

> 📝 **Draft.** This chapter has not been reviewed yet — content may be incomplete or change.

> **Phase 3 of the [lending walkthrough](./lending-protocol.md).** The offer is
> open. A lender accepts it with `SetupLending`, or the borrower withdraws it with
> `CancelOffer` — the two spending paths of the one `pre_lock` covenant.

[Opening the offer](./lending-offer.md) left the collateral and NFTs behind the
`pre_lock` covenant in state `offer_open`. That covenant can be spent two ways, and
this phase is where [multiple spending paths](../recipes/06-multiple-spending-paths.md)
and the [ScriptAuth wrapper](../recipes/05-covenant-utxo-types.md) carry real
weight.

## Two paths, selected by a witness

`pre_lock` accepts two spending paths, and the manifest exposes each as its own
method: `SetupLending` takes the **accept** path, `CancelOffer` the **cancel**
path. A method picks its path with a `PATH` witness — `Left(())` or `Right(())` —
the `simplicityhl` selector from
[Multiple spending paths](../recipes/06-multiple-spending-paths.md). How each path
is enforced on-chain is out of scope here; what's new for the manifest are two
witness patterns this phase relies on, both visible in the JSON below:

- **`SPEND_PATH`** — a `taproot_leaf` witness on every covenant input, naming which
  tapleaf is being spent. Its value comes from a `…_leaf` formula (e.g.
  `pre_lock_leaf`).
- **`INPUT_SCRIPT_INDEX`** — a `simplicityhl` witness on each NFT input. The NFTs
  live in `prelock_script_auth` UTXOs (a
  [`script_auth`](../recipes/05-covenant-utxo-types.md) covenant type that must be
  co-spent with the collateral); the witness just tells that covenant which input
  the collateral is at — here, `0`.

## `SetupLending` — the lender accepts (`PATH::LEFT`)

The lender spends the `pre_lock` collateral via the accept path (`PATH = Left(())`),
supplies the principal, and produces the active loan: collateral into the `lending`
covenant, principal to the borrower, the NFTs re-wrapped under the lending-phase
`script_auth`, and the Lender NFT to the lender's wallet.

The covenant requires those inputs and outputs in an **exact order** — and because
the principal is injected at output 1, the NFT outputs sit one slot below their
inputs. Rather than hope the builder lands on that layout, every input and output
declares an explicit `required_index`:

```json
"SetupLending": {
  "inputs": [
    { "id": "collateral_in",   "utxo_source": { "utxo_type": "pre_lock" }, "required_index": 0,
      "witnesses": { "PATH": { "type": "simplicityhl", "simplicity_type": "Either<()>", "value": "Left(())" },
                     "SPEND_PATH": { "type": "taproot_leaf", "source": { "type": "formula", "expr": "pre_lock_leaf" } } } },
    { "id": "first_params_in",  "utxo_source": { "utxo_type": "prelock_script_auth" }, "required_index": 1,
      "witnesses": { "INPUT_SCRIPT_INDEX": { "type": "simplicityhl", "simplicity_type": "u32", "value": "0" },
                     "SPEND_PATH": { "type": "taproot_leaf", "source": { "type": "formula", "expr": "prelock_script_auth_leaf" } } } },
    { "id": "second_params_in", "utxo_source": { "utxo_type": "prelock_script_auth" }, "required_index": 2, "…": "…" },
    { "id": "borrower_nft_in",  "utxo_source": { "utxo_type": "prelock_script_auth" }, "required_index": 3, "…": "…" },
    { "id": "lender_nft_in",    "utxo_source": { "utxo_type": "prelock_script_auth" }, "required_index": 4, "…": "…" },
    { "id": "principal_in",     "utxo_source": "wallet", "asset": "compile_params.PRINCIPAL_ASSET_ID", "required_index": 5,
      "amount_sat": { "min_amount": "compile_params.PRINCIPAL_AMOUNT" } },
    { "id": "fee_input", "utxo_source": "wallet", "asset": "lbtc", "optional": true, "required_index": 6 }
  ],
  "outputs": [
    { "id": "lending_collateral_out", "destination": { "utxo_type": "lending_collateral" },  "required_index": 0, "asset": "compile_params.COLLATERAL_ASSET_ID", "amount_sat": "compile_params.COLLATERAL_AMOUNT" },
    { "id": "principal_to_borrower",  "destination": { "utxo_type": "p2pk" },                "required_index": 1, "asset": "compile_params.PRINCIPAL_ASSET_ID",  "amount_sat": "compile_params.PRINCIPAL_AMOUNT" },
    { "id": "first_params_relocked",  "destination": { "utxo_type": "lending_script_auth" }, "required_index": 2, "…": "…" },
    { "id": "second_params_relocked", "destination": { "utxo_type": "lending_script_auth" }, "required_index": 3, "…": "…" },
    { "id": "borrower_nft_released",  "destination": { "utxo_type": "lending_script_auth" }, "required_index": 4, "asset": "compile_params.BORROWER_NFT_ASSET_ID", "amount_sat": 1 },
    { "id": "lender_nft_released",    "destination": "wallet",                                "required_index": 5, "asset": "compile_params.LENDER_NFT_ASSET_ID", "amount_sat": 1 },
    { "id": "principal_change", "destination": "change", "asset": "compile_params.PRINCIPAL_ASSET_ID", "optional": true, "required_index": -2 },
    { "id": "fee_change",       "destination": "change", "asset": "lbtc", "optional": true, "required_index": -1 }
  ]
}
```

Three details that make this work:

- **`required_index` is the contract between manifest and covenant.** The covenant
  reads `output_amount(2)` and `output_script_hash(0)` by literal index; if the
  builder placed them anywhere else the on-chain check fails. Negative indices
  (`-1`, `-2`) pin the optional change outputs to the *end*, so they never disturb
  the fixed prefix. This is the discipline [recipe 6](../recipes/06-multiple-spending-paths.md)
  flags as essential for introspecting covenants.
- **The Lender NFT goes to the lender's wallet** (output 5, `destination: wallet`),
  not back into a covenant. It's now the lender's bearer claim — they'll need it to
  liquidate or to drain the vault in [settlement](./lending-settlement.md).
- **The NFTs re-wrap to a different `utxo_type`.** Outputs 2–4 target
  `lending_script_auth` instead of `prelock_script_auth` — the same `script_auth`
  program compiled to `LENDING_COV_HASH` rather than `PRE_LOCK_COV_HASH`. A
  destination's `utxo_type` is all the manifest needs to move the tokens from the
  offer-phase covenant to the active-loan one.

## `CancelOffer` — the borrower backs out (`PATH::RIGHT`)

If no lender accepts, the borrower reclaims the collateral and destroys the offer.
`CancelOffer` takes the cancel path (`PATH = Right(())`), which the covenant gates
with a borrower signature — so this method adds a `SIGNATURE` witness sourced from
`BORROWER_PUB_KEY` — and routes every NFT to an `op_return` to burn it. Here the
outputs line up one-to-one with the inputs (no principal is injected), so no
`required_index` is needed; the collateral returns to the borrower's wallet:

```json
"CancelOffer": {
  "inputs": [
    { "id": "pre_lock_in", "utxo_source": { "utxo_type": "pre_lock" },
      "witnesses": { "PATH": { "type": "simplicityhl", "value": "Right(())" },
                     "SIGNATURE": { "type": "Signature", "sig_type": "sig_hash_all",
                                    "source": { "type": "wallet", "key": "compile_params.BORROWER_PUB_KEY" } },
                     "SPEND_PATH": { "type": "taproot_leaf", "source": { "type": "formula", "expr": "pre_lock_leaf" } } } },
    { "id": "first_params_in", "utxo_source": { "utxo_type": "prelock_script_auth" }, "…": "…" },
    "…borrower & lender NFTs…",
    { "id": "fee_input", "utxo_source": "wallet", "asset": "lbtc" }
  ],
  "outputs": [
    { "id": "collateral_returned", "destination": "wallet", "asset": "compile_params.COLLATERAL_ASSET_ID", "amount_sat": "pre_lock_in.amount_sat" },
    { "id": "first_params_burned",  "destination": { "type": "op_return" }, "asset": "compile_params.FIRST_PARAMETERS_NFT_ASSET_ID",  "amount_sat": "first_params_in.amount_sat" },
    { "id": "second_params_burned", "destination": { "type": "op_return" }, "asset": "compile_params.SECOND_PARAMETERS_NFT_ASSET_ID", "amount_sat": "second_params_in.amount_sat" },
    { "id": "borrower_nft_burned",  "destination": { "type": "op_return" }, "asset": "compile_params.BORROWER_NFT_ASSET_ID", "amount_sat": 1 },
    { "id": "lender_nft_burned",    "destination": { "type": "op_return" }, "asset": "compile_params.LENDER_NFT_ASSET_ID",   "amount_sat": 1 },
    { "id": "fee_change", "destination": "change", "asset": "lbtc", "optional": true }
  ]
}
```

`CancelOffer` is **unilateral** in the lifecycle — the borrower needs no one's
cooperation, exactly as a trustless escape hatch should be. It mirrors the cold-key
break-out from the [Last Will](../recipes/04b-last-will.md): a signature-gated path
that ends the contract.

> **Test the cancel path too.** The happy flow runs `SetupLending`; `CancelOffer`
> exercises the *other* `pre_lock` branch. If you only verify acceptance, the
> cancel path stays untested — run it against a fresh open offer separately.

## Run it

Accepting is run by the **lender**; cancelling by the **borrower**. Both need the
instance file the borrower wrote at construction.

For the lender to accept, they need a single UTXO holding exactly the principal.
`PrepareLender` carves one off any larger UTXO of the principal asset; then
`SetupLending` does the handshake:

```sh
# --- Lender accepts ---
txw run examples/lending/txmanifest.json PrepareLender --wallet lender.json
txw run examples/lending/txmanifest.json SetupLending \
  --instance examples/lending/lending.instance.json \
  --state examples/lending/lending.state.json \
  --wallet lender.json
txw sync --wallet lender.json
```

To exercise the other branch instead, the **borrower** runs `CancelOffer` (any
time before a lender accepts) and gets the collateral back, burning the NFTs:

```sh
txw run examples/lending/txmanifest.json CancelOffer \
  --instance examples/lending/lending.instance.json --wallet borrower.json
```

> **Both wallets need the instance file.** `SetupLending` is run by the *lender*,
> but it still takes `--instance lending.instance.json` — the file the *borrower*
> produced at construction. The lender needs it to rebuild the covenant addresses
> and the packed terms. In a real deployment the borrower publishes the instance
> (or an indexer reconstructs it from the on-chain NFTs and the `OP_RETURN`
> beacon); here, share the file between the two wallets.

Once `SetupLending` broadcasts, the loan is `loan_active`: the borrower has the
principal and the collateral is held by the `lending` covenant. On to
**[settling the loan](./lending-settlement.md)**.
