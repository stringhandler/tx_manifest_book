# Issuing the NFTs & encoding the terms

> 📝 **Draft.** This chapter has not been reviewed yet — content may be incomplete or change.

> **Phase 1 of the [lending walkthrough](./lending-protocol.md).** The borrower
> constructs a loan offer: mint four NFTs, pack the loan terms into two of them,
> compute the covenant addresses the rest of the protocol locks to, and write it
> all into an instance file.

Nothing is on-chain *as a loan* yet after this step — `IssueUtilityNFTs` only
mints the tokens and records the parameters. But it's the densest action in the
protocol, because it's where four cookbook ideas converge:

- **Issuance** — minting new Liquid assets whose IDs come from input outpoints
  ([recipe 8](../recipes/08-issuance-and-nfts.md)).
- **Bit-packing** — encoding the loan terms into NFT *amount* fields so the
  covenants can read them on-chain ([recipe 7](../recipes/07-formulas-and-derived-params.md)).
- **Tapleaf compute** — deriving each covenant's script hash by compiling a
  `.simf`, where some hashes feed into others ([recipe 9](../recipes/09-hooks-and-tapleaf.md)).
- **A constructor** — `create_instance` persists the whole deal to a file so every
  later method can read it back ([recipe 10](../recipes/10-instance-state-constructors.md)).

## The constructor and its terms

`IssueUtilityNFTs` is the class's `is_constructor` method. Its `params` are the
loan terms the borrower chooses:

```json
"IssueUtilityNFTs": {
  "is_constructor": true,
  "params": {
    "BORROWER_PUB_KEY":            { "type": "pubkey", "source": { "type": "wallet_key" } },
    "COLLATERAL_ASSET_ID":         { "type": "liquid.asset_id" },
    "COLLATERAL_AMOUNT":           { "type": "u64" },
    "COLLATERAL_DECIMALS_MANTISSA":{ "type": "u8", "default": "8" },
    "PRINCIPAL_ASSET_ID":          { "type": "liquid.asset_id" },
    "PRINCIPAL_AMOUNT":            { "type": "u64" },
    "PRINCIPAL_DECIMALS_MANTISSA": { "type": "u8" },
    "PRINCIPAL_INTEREST_RATE":     { "type": "u16" },
    "LOAN_EXPIRATION_TIME":        { "type": "u32" }
  },
  ...
}
```

`BORROWER_PUB_KEY` auto-fills from the borrower's wallet signing key (the
`wallet_key` source from [Parameters](../recipes/02-params-and-validations.md)).
The interest rate is in **basis points** (`u16`, so 10,000 = 100%); the
expiry is a **block height** (CLTV). The two `DECIMALS_MANTISSA` values matter for
the encoding below — they let a base-10 amount like "1 L-BTC" be stored compactly
as `1` with a separate exponent of `8`.

## Minting four NFTs from issuance inputs

> **New to issuance?** This is the first action in the book that mints assets.
> [Asset issuance & NFTs](../recipes/08-issuance-and-nfts.md) covers the
> `issuance` block, why asset IDs come from outpoints, and bearer-token NFTs in
> isolation — read it first if any of the below is unfamiliar.

A Liquid asset ID is derived from the **outpoint** of the input that issues it, so
to mint four distinct NFTs you need four distinct wallet UTXOs. That's why the
helper `Prepare` action splits one UTXO into four beforehand. Each issuance input
declares an `issuance` block and captures the resulting asset ID with an
`on_resolved` hook:

```json
{
  "id": "borrower_nft_issuance_input",
  "utxo_source": "wallet",
  "asset": "lbtc",
  "issuance": { "kind": "new", "asset_amount_sat": 1, "inflation_amount_sat": 0 },
  "on_resolved": { "set": { "compile_params.BORROWER_NFT_ASSET_ID": "asset" } }
}
```

The Borrower and Lender NFTs are issued with `asset_amount_sat: 1` — true
single-unit bearer tokens. The **two Parameter NFTs are different**: their issued
amount is not `1` but the *encoded loan terms* (next section), so the asset's very
supply carries the offer:

```json
{
  "id": "first_params_issuance_input",
  "utxo_source": "wallet",
  "asset": "lbtc",
  "issuance": { "kind": "new", "asset_amount_sat": "compile_params.FIRST_PARAMETERS_ENCODED", "inflation_amount_sat": 0 },
  "on_resolved": { "set": { "compile_params.FIRST_PARAMETERS_NFT_ASSET_ID": "asset" } }
}
```

All four asset IDs land in `compile_params.*` via `on_resolved`, ready for the
covenant-hash computation. The matching `outputs` simply send each freshly minted
NFT back to the borrower's wallet. See
[Asset issuance & NFTs](../recipes/08-issuance-and-nfts.md) for the issuance
mechanics in isolation.

## Bit-packing the loan terms

Here's the trick that makes the offer self-describing on-chain. Rather than store
the terms off-chain, the protocol packs them into the **amount fields** of the two
Parameter NFTs, computed in an `on_pre_broadcast` hook before the transaction is
built:

```json
"on_pre_broadcast": {
  "set": {
    "compile_params.FIRST_PARAMETERS_ENCODED":
      "params.PRINCIPAL_INTEREST_RATE + params.LOAN_EXPIRATION_TIME * 65536 + params.COLLATERAL_DECIMALS_MANTISSA * 8796093022208 + params.PRINCIPAL_DECIMALS_MANTISSA * 140737488355328",
    "compile_params.SECOND_PARAMETERS_ENCODED":
      "params.COLLATERAL_AMOUNT / pow(10, COLLATERAL_DECIMALS_MANTISSA) + params.PRINCIPAL_AMOUNT / pow(10, PRINCIPAL_DECIMALS_MANTISSA) * 33554432"
  }
}
```

Each multiplier is a power of two — it's a left-shift dressed up as multiplication.
The **First Parameters** amount packs four fields into a 64-bit integer:

| Field | Bits | Width | Multiplier |
|-------|------|-------|------------|
| `PRINCIPAL_INTEREST_RATE` | 0–15 | 16 | ×1 |
| `LOAN_EXPIRATION_TIME` | 16–42 | 27 | ×65536 (2¹⁶) |
| `COLLATERAL_DECIMALS_MANTISSA` | 43–46 | 4 | ×8796093022208 (2⁴³) |
| `PRINCIPAL_DECIMALS_MANTISSA` | 47–50 | 4 | ×140737488355328 (2⁴⁷) |

The **Second Parameters** amount packs the two *base* amounts — each divided down
by its decimal exponent so it fits in 25 bits:

| Field | Bits | Width | Multiplier |
|-------|------|-------|------------|
| `COLLATERAL_AMOUNT / 10^collateral_decimals` | 0–24 | 25 | ×1 |
| `PRINCIPAL_AMOUNT / 10^principal_decimals` | 25–49 | 25 | ×33554432 (2²⁵) |

This is exactly the layout the covenants *unpack*. In
[`pre_lock.simf`](https://github.com/stringhandler/txmanifest-wallet/blob/main/examples/lending/pre_lock.simf)
and `lending.simf`, `extract_lending_parameters` masks and shifts these same bit
ranges back out, then `validate_lending_params` asserts they match the covenant's
own compile params:

```rust
let (interest_rate_raw, shift): (u64, u8) =
    extract_bits_from_amount(first_parameters_amount, 16, 0);   // bits 0–15
let (loan_expiration_time_raw, shift): (u64, u8) =
    extract_bits_from_amount(first_parameters_amount, 27, shift); // bits 16–42
// …decimals…  then from the second NFT, two 25-bit base amounts
```

The manifest's packing and the covenant's unpacking are two halves of one wire
format — get the widths or multipliers out of sync and `validate_lending_params`
aborts the spend. That mutual dependence is the whole reason the encoding lives in
the example as a worked reference rather than something you'd reinvent.

> **Why pack at all?** A covenant can only introspect what's *in the transaction*.
> By making the terms the NFTs' amounts, every spend that moves the NFTs carries
> the terms with it, and each covenant re-derives and re-checks them — no oracle,
> no side channel. The cost is the 64-bit budget you're packing into, which is why
> base amounts are stored with a separate decimals exponent.

## Computing the covenant addresses

The borrower has to lock collateral to the `pre_lock` covenant — but `pre_lock`'s
address depends on the `lending` covenant's hash, which depends on the principal
*vault's* hash, which depends on the Lender NFT's asset ID, which only exists
*after* the issuance inputs resolve. `create_instance` untangles this with a set
of **tapleaf compute** fields, each compiling a `.simf` to its script hash:

```json
"create_instance": {
  "class": "lending_contract",
  "fields": {
    "PRINCIPAL_OUTPUT_SCRIPT_HASH": {
      "compute": "tapleaf", "simf": "./p2pk.simf",
      "params": { "PUB_KEY": { "type": "pubkey", "value": "BORROWER_PUB_KEY" } }
    },
    "LENDER_PRINCIPAL_COV_HASH": {
      "compute": "tapleaf", "simf": "./asset_auth.simf",
      "params": {
        "ASSET_ID":        { "type": "liquid.asset_id", "value": "LENDER_NFT_ASSET_ID" },
        "ASSET_AMOUNT":    { "type": "u64",  "value": "1" },
        "WITH_ASSET_BURN": { "type": "bool", "value": "true" }
      }
    },
    "LENDING_COV_HASH": {
      "compute": "tapleaf", "simf": "./lending.simf",
      "params": { "…": "…", "LENDER_PRINCIPAL_COV_HASH": { "type": "bytes32", "value": "LENDER_PRINCIPAL_COV_HASH" } }
    },
    "PRE_LOCK_COV_HASH": {
      "compute": "tapleaf", "simf": "./pre_lock.simf",
      "params": { "…": "…", "LENDING_COV_HASH": { "type": "bytes32", "value": "LENDING_COV_HASH" } }
    },
    "…": "…"
  }
}
```

Read the `value` strings as references to *other fields already computed*. The
dependency order forms a chain, not a cycle:

```text
  p2pk.simf ───────────────► PRINCIPAL_OUTPUT_SCRIPT_HASH ─┐
  asset_auth.simf ─────────► LENDER_PRINCIPAL_COV_HASH ─┐  │
                                                        ▼  │
  lending.simf  ──────────►  LENDING_COV_HASH ──┬──────────┤
                                                │          │
  script_auth.simf(LENDING) ► PARAMETERS_NFT_OUTPUT_SCRIPT_HASH,
                              BORROWER_NFT_OUTPUT_SCRIPT_HASH ─┐
                                                               ▼
  pre_lock.simf ───────────► PRE_LOCK_COV_HASH ◄───────────────┘
  script_auth.simf(PRE_LOCK) ► PRELOCK_PARAMETERS_NFT_SCRIPT_HASH
```

The tool compiles them in dependency order: leaf programs first (`p2pk`,
`asset_auth`), then `lending` (which needs the vault hash), then the `script_auth`
wrappers keyed to `LENDING_COV_HASH`, and finally `pre_lock` (which needs all of
the above) and its own `script_auth` wrapper.

> **`script_auth.simf` compiled twice.** The same program appears as two distinct
> UTXO types — `prelock_script_auth` and `lending_script_auth` — because it's
> compiled with two different `SCRIPT_HASH` params. One wraps the NFTs to the
> `pre_lock` covenant during the offer; the other re-wraps them to the `lending`
> covenant once the loan is active. Same code, two addresses. The
> [tapleaf compute](../recipes/09-hooks-and-tapleaf.md) recipe covers this pattern;
> [covenant UTXO types](../recipes/05-covenant-utxo-types.md) covers why a `.simf`
> plus its params is an address.

## One derived field: the interest amount

Most instance fields are either passthrough (`"$params.X"`) or tapleaf hashes. One
is a plain arithmetic [derived param](../recipes/07-formulas-and-derived-params.md):

```json
"PRINCIPAL_INTEREST_AMOUNT": "params.PRINCIPAL_AMOUNT * params.PRINCIPAL_INTEREST_RATE / 10000"
```

The borrower never enters the interest *amount* — only the *rate*. The actual
satoshis owed are computed once, here, and stored in the instance so `RepayLoan`
can require exactly principal + interest later. (The `lending` covenant computes
the same figure on-chain in `calculate_interest`, so the two agree.)

## What you end up with

After broadcast, `create_instance` writes
`lending.instance.json` next to the manifest, holding every field above: the four
asset IDs, the four covenant hashes, the packed parameter values, the interest
amount, and the borrower's key. **This file is the deal.** Every later
method — `LockCollateral`, `SetupLending`, `RepayLoan`, and the rest — is run with
`--instance lending.instance.json` so the wallet rebuilds the exact same covenant
addresses without re-entering anything. This is the full class / instance model
from [Instance, state & constructors](../recipes/10-instance-state-constructors.md).

## Run it

`IssueUtilityNFTs` needs four separate L-BTC UTXOs (one per issuance). `Prepare`
splits one into four; then run the constructor as the **borrower**:

```sh
# split one wallet UTXO into four for the four issuances
txw run examples/lending/txmanifest.json Prepare \
  --wallet borrower.json
```

### A testnet params file

`IssueUtilityNFTs` would otherwise prompt for every loan term. Drop a
[params file](../recipes/04b-last-will.md#skip-the-prompts-a-params-file) next to
the manifest and the tool auto-discovers it — for `testnet` it looks for
`txmanifest.testnet.json` in `examples/lending/`. Here's a complete one for a
**single-asset L-BTC loan** (borrow testnet L-BTC against testnet L-BTC), so the
[faucet](https://liquidtestnet.com/faucet) can fund both the borrower and the
lender:

```json
{
  "COLLATERAL_ASSET_ID":          "144c654344aa716d6f3abcc1ca90e5641e4e2a7f633bc09fe3baf64585819a49",
  "COLLATERAL_AMOUNT":            "200000",
  "COLLATERAL_DECIMALS_MANTISSA": "0",
  "PRINCIPAL_ASSET_ID":           "144c654344aa716d6f3abcc1ca90e5641e4e2a7f633bc09fe3baf64585819a49",
  "PRINCIPAL_AMOUNT":             "100000",
  "PRINCIPAL_DECIMALS_MANTISSA":  "0",
  "PRINCIPAL_INTEREST_RATE":      "1000",
  "LOAN_EXPIRATION_TIME":         "5000000"
}
```

That id is testnet L-BTC (the same asset the
[faucet](https://liquidtestnet.com/faucet) dispenses). The terms describe a
0.002 L-BTC collateral loan for 0.001 L-BTC principal at 10% interest
(`1000` basis points). A few values are worth understanding rather than copying
blindly:

- **`BORROWER_PUB_KEY` is absent on purpose** — it's a `wallet_key` source, so it
  auto-fills from the borrower wallet. The file only carries the terms you choose.
- **`*_DECIMALS_MANTISSA` is `0` here, not `8`.** Recall from
  [bit-packing](#bit-packing-the-loan-terms) that each amount is stored as
  `amount / 10^decimals` in a **25-bit** base field, and the covenant rebuilds it
  as `base × 10^decimals`. So the amount must be an exact multiple of `10^decimals`
  *and* the base must fit in 25 bits (< ~33.5M). With `decimals = 0` the raw
  satoshi amount goes straight into the base field — perfect for sub-`0.335`-L-BTC
  testnet sums. For whole-coin amounts you'd raise the exponent (e.g. `8`, the
  manifest's default) so larger figures still fit the 25 bits — but then the amount
  must be a whole multiple of `10^8` (≥ 1 L-BTC), which the faucet won't cover.
- **`LOAN_EXPIRATION_TIME` is a block height** — set it comfortably *ahead* of the
  current testnet tip (check an explorer; `5000000` is a placeholder). It only
  matters at [liquidation](./lending-settlement.md#liquidateafterexpiry--lender-seizes-pathright);
  the constructor just records it.

With the file in place, the constructor reads every value from it — nothing to
type:

```sh
# mint the NFTs, encode the terms, compute hashes, write the instance file
txw run examples/lending/txmanifest.json IssueUtilityNFTs \
  --network testnet --wallet borrower.json
```

On success the four NFTs are in the borrower's wallet and
`examples/lending/lending.instance.json` exists. Sync, and confirm the NFTs:

```sh
txw sync --wallet borrower.json
txw get-balance --wallet borrower.json   # four new single-asset balances
```

### Inspect the instance file

Open `examples/lending/lending.instance.json` to see what the constructor
recorded — this is what every later method reads back:

```json
{
  "instance": {
    "class": "lending_contract",
    "fields": {
      "BORROWER_NFT_ASSET_ID": "f94aff7f54bd4f4076a0aa07635264a32926966e119dc523ac86427d1f2239f7",
      "BORROWER_NFT_OUTPUT_SCRIPT_HASH": "c4b8e6299c4924f9375650c24457a3cb6c69c54cf66afcd0f8b667146ce55667",
      "BORROWER_PUB_KEY": "0b9fa04ada4fcaa83b148ae76fee98fa1bd3a84a1eefe42295e0b98e1fbcac72",
      "COLLATERAL_AMOUNT": "3452",
      "COLLATERAL_ASSET_ID": "144c654344aa716d6f3abcc1ca90e5641e4e2a7f633bc09fe3baf64585819a49",
      "COLLATERAL_DECIMALS_MANTISSA": "0",
      "FIRST_PARAMETERS_ENCODED": "327680000100",
      "FIRST_PARAMETERS_NFT_ASSET_ID": "c59e58652d6a00ba53e2ef97556210b106aff996fd76a8de8d04bcbef6888775",
      "LENDER_NFT_ASSET_ID": "0d51f8bcf2f6fe4e5c6ebc886a7cc87323e8599c5652ddb44bb6ac6c2e690d52",
      "LENDER_PRINCIPAL_COV_HASH": "279a4424550ccc694525388d9c461166a1454defadff4204d1a0885bd5ca2b83",
      "LENDING_COV_HASH": "135596e46be4b2a229ae25fea585b056e5b909e0aaee87fd6fc785c9201dd6cf",
      "LOAN_EXPIRATION_TIME": "5000000",
      "PARAMETERS_NFT_OUTPUT_SCRIPT_HASH": "c4b8e6299c4924f9375650c24457a3cb6c69c54cf66afcd0f8b667146ce55667",
      "PRELOCK_PARAMETERS_NFT_SCRIPT_HASH": "761c8268a5998d892c41df708dd64c18047b54750a760861af88e68336c1cfea",
      "PRE_LOCK_COV_HASH": "4ac452b2c2c79b932d74f7fee114106328ce18d68ad92091ed345c6c22f59f07",
      "PRINCIPAL_AMOUNT": "1000",
      "PRINCIPAL_ASSET_ID": "144c654344aa716d6f3abcc1ca90e5641e4e2a7f633bc09fe3baf64585819a49",
      "PRINCIPAL_DECIMALS_MANTISSA": "0",
      "PRINCIPAL_INTEREST_AMOUNT": "10",
      "PRINCIPAL_INTEREST_RATE": "100",
      "PRINCIPAL_OUTPUT_SCRIPT_HASH": "b4966bd0290ef509e7b1a98dd0b3fe54bd1860f9bb163bf3acb3d3a8401d41d4",
      "SECOND_PARAMETERS_ENCODED": "33554435452",
      "SECOND_PARAMETERS_NFT_ASSET_ID": "5a001563b2384198a09127c3ac6f36d0cc2a980000fd9915c3ca3cc2d6e2c136"
    }
  }
}
```

What to look at:

- **The four `*_ASSET_ID` fields** are your freshly minted NFTs — derived from the
  issuance [outpoints](../recipes/08-issuance-and-nfts.md#why-the-asset-id-comes-from-the-outpoint),
  so they're unique to this run.
- **The four covenant hashes** (`PRE_LOCK_COV_HASH`, `LENDING_COV_HASH`,
  `LENDER_PRINCIPAL_COV_HASH`, and the `*_SCRIPT_HASH` wrappers) are the addresses
  the next steps lock to — computed from those asset IDs and your `BORROWER_PUB_KEY`.
- **`FIRST_PARAMETERS_ENCODED` / `SECOND_PARAMETERS_ENCODED`** are the
  [bit-packed](#bit-packing-the-loan-terms) terms — and also the amounts your two
  Parameter NFTs were issued with. You can check them by hand: with the decimal
  fields `0`, `FIRST = INTEREST_RATE + EXPIRY × 65536 = 100 + 5000000 × 65536 =
  327680000100`, and `SECOND = COLLATERAL + PRINCIPAL × 33554432 = 3452 + 1000 ×
  33554432 = 33554435452`.

> **Yours will differ.** Almost every value is derived, so a fresh run won't
> reproduce these — the asset IDs come from your outpoints and the hashes from your
> key. This sample also happens to come from a run with *different* terms than the
> params file above (a `3452`-sat collateral loan at 1% interest), so the amounts
> won't match either. It's here to show the shape and what each field is for.

With the offer constructed, the borrower can put it on-chain:
**[Opening the offer](./lending-offer.md)**.
