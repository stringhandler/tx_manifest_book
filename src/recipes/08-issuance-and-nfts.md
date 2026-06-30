# Asset issuance & NFTs

> **Problem.** Mint a brand-new Liquid asset — including a single-unit NFT — as
> part of an action, and use its derived asset ID in the same transaction and in
> later ones.

Every recipe so far has *moved* assets that already existed (L-BTC, a covenant's
collateral). This one **creates** them. On Liquid, an asset is issued as a property
of a transaction input: you point at a wallet UTXO, attach an `issuance` block, and
the transaction mints a new asset whose ID is derived from that input. Two ideas do
most of the work:

1. The new asset's **ID comes from the outpoint** of the issuing input — so it's
   unique and unforgeable, but unknown until the input is chosen.
2. You **capture that ID** with an `on_resolved` hook so the rest of the action
   (and the instance file) can refer to it.

The only example manifest that issues assets is the lending protocol, so the
snippets below are drawn from its `IssueUtilityNFTs` constructor — reduced to one
asset at a time. The [lending walkthrough](../walkthrough/lending-issuance.md)
shows all four issuances together.

## The `issuance` block

Add `issuance` to any wallet input to mint an asset as that input is spent:

```json
{
  "id": "nft_issuance_input",
  "utxo_source": "wallet",
  "asset": "lbtc",
  "issuance": { "kind": "new", "asset_amount_sat": 1, "inflation_amount_sat": 0 }
}
```

| Field | Required | Purpose |
|-------|----------|---------|
| `kind` | yes | `"new"` for a first issuance, `"reissue"` to mint more of an existing asset. |
| `asset_amount_sat` | yes | How many units to issue, in the asset's base denomination. A literal or a [formula](./07-formulas-and-derived-params.md). |
| `inflation_amount_sat` | no | Inflation (reissuance-token) amount. `0` disables reissuance — the supply is fixed forever. |

The input is still an ordinary input: it's a wallet UTXO you also spend for its
L-BTC (here, to pay the fee). The `issuance` block just rides along on it.

## Why the asset ID comes from the outpoint

A Liquid asset ID is computed from the **outpoint** (txid + vout) of the input that
issues it. That makes the ID globally unique without a registry — no two inputs can
ever share an outpoint — but it has a practical consequence:

> **One issuance per input, and the ID isn't known up front.** To mint *N* distinct
> assets in one transaction you need *N* distinct input UTXOs. And because the ID
> depends on which UTXO the wallet picks, you can't hard-code it — you compute it
> during the build and capture it (next section).

This is why the lending protocol ships a `Prepare` helper action that splits one
wallet UTXO into four before `IssueUtilityNFTs` runs: four NFTs need four separate
inputs to issue from.

## Capturing the new asset ID with `on_resolved`

Once the build picks the input's UTXO, its outpoint — and therefore the new asset
ID — is fixed. An `on_resolved` hook on the input fires at that moment and lets you
stash the ID into a compile param:

```json
{
  "id": "nft_issuance_input",
  "utxo_source": "wallet",
  "asset": "lbtc",
  "issuance": { "kind": "new", "asset_amount_sat": 1, "inflation_amount_sat": 0 },
  "on_resolved": { "set": { "compile_params.BORROWER_NFT_ASSET_ID": "asset" } }
}
```

Inside an input's own `on_resolved`, the bare word **`asset`** means *this input's*
resolved asset ID — the freshly minted one. Elsewhere in the action you refer to it
by the input's id, as **`nft_issuance_input.asset`** (the general
[formula reference](./07-formulas-and-derived-params.md) form). From here on,
`compile_params.BORROWER_NFT_ASSET_ID` is a normal param: you can lock outputs to
it, feed it into a covenant's compile params, or write it into the instance file.

> **Hooks recap.** `on_resolved` runs per-input as soon as that input's UTXO is
> known; `on_pre_broadcast` runs once per action just before building. Both run
> `set` assignments. See [Hooks & tapleaf compute](./09-hooks-and-tapleaf.md).

## Run it

Issuance has no standalone example manifest — it's exercised by the lending
constructor. `IssueUtilityNFTs` needs four separate L-BTC UTXOs (one per
issuance), which `Prepare` carves out first:

```sh
txw run examples/lending/txmanifest.json Prepare \
  --wallet borrower.json
txw run examples/lending/txmanifest.json IssueUtilityNFTs \
  --wallet borrower.json
txw sync --wallet borrower.json
txw get-balance --wallet borrower.json   # four new single-asset balances
```

After it broadcasts, the wallet holds four newly minted assets and the instance
file records their IDs. To see the transaction's outputs (including the issuance
outputs) before broadcasting, add `--export-pset issue.pset.json` and decode it.

## Try next

You've minted assets and captured their derived IDs. The full four-NFT
construction — plus packing loan terms into amounts and computing the covenants
those NFTs get locked to — is the first phase of the lending walkthrough:
[Issuing the NFTs & encoding the terms](../walkthrough/lending-issuance.md). The
hooks that capture and derive these values get their own recipe:
[Hooks & tapleaf compute](./09-hooks-and-tapleaf.md).
