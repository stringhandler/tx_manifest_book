# The lending protocol

> 📝 **Draft.** This chapter has not been reviewed yet — content may be incomplete or change.

> **Problem.** Two strangers want to transact a collateralised loan with no
> escrow agent and no trusted backend. A *borrower* locks collateral and
> advertises terms; a *lender* supplies the principal; the loan later settles by
> repayment or — if the borrower defaults — by liquidation after a deadline.
> Every rule is enforced on-chain by Simplicity covenants.

This is the capstone of the cookbook. Everything the recipes introduced one at a
time — [covenant UTXO types](../recipes/05-covenant-utxo-types.md),
[multiple spending paths](../recipes/06-multiple-spending-paths.md),
[asset issuance & NFTs](../recipes/08-issuance-and-nfts.md),
[formulas & derived params](../recipes/07-formulas-and-derived-params.md),
[hooks & tapleaf compute](../recipes/09-hooks-and-tapleaf.md), and the
[class / instance model](../recipes/10-instance-state-constructors.md) — shows up
here at once, wired into a single working protocol.

The full example lives in the repository at
[`examples/lending/`](https://github.com/stringhandler/txmanifest-wallet/tree/main/examples/lending):
one [`txmanifest.json`](https://github.com/stringhandler/txmanifest-wallet/blob/main/examples/lending/txmanifest.json)
plus five `.simf` covenant programs. This chapter walks through it in the order
you'd actually run it.

## The deal, in one paragraph

Alice has collateral (say, L-BTC) and wants to borrow L-USDT against it without
selling. She locks her collateral into a covenant and publishes her terms —
amount, interest, expiry — as on-chain NFTs anyone can read. Bob sees the offer,
likes the terms, and accepts by sending Alice the principal; in the same
transaction her collateral moves into a second covenant that holds it for the
life of the loan. To get her collateral back, Alice repays principal plus
interest before the deadline. If she doesn't, Bob can seize the collateral once
the deadline passes. At no point does either party have to trust the other or any
third party — the covenants only permit the honest transitions.

## The cast

**Two roles.** The protocol has a **borrower** and a **lender**. They are
different people with different wallets, so when you run it you'll keep two wallet
files — `borrower.json` and `lender.json` — and run each action as the
appropriate party.

**Five covenant programs.** Each `.simf` file is a small Simplicity program that
gates one kind of UTXO. They are deliberately tiny and composable:

| Program | Role |
|---------|------|
| [`pre_lock.simf`](https://github.com/stringhandler/txmanifest-wallet/blob/main/examples/lending/pre_lock.simf) | Holds the collateral **while the offer is open**. `PATH::LEFT` lets a lender activate the loan; `PATH::RIGHT` lets the borrower cancel (with a signature). |
| [`lending.simf`](https://github.com/stringhandler/txmanifest-wallet/blob/main/examples/lending/lending.simf) | Holds the collateral **during the active loan**. `PATH::LEFT` is repayment; `PATH::RIGHT` is liquidation after expiry. |
| [`script_auth.simf`](https://github.com/stringhandler/txmanifest-wallet/blob/main/examples/lending/script_auth.simf) | Wraps each NFT so it can only be spent **co-spent with** the right collateral covenant. The glue that binds the NFTs to the deal. |
| [`asset_auth.simf`](https://github.com/stringhandler/txmanifest-wallet/blob/main/examples/lending/asset_auth.simf) | Guards the lender's **principal vault**: release requires burning the Lender NFT. |
| [`p2pk.simf`](https://github.com/stringhandler/txmanifest-wallet/blob/main/examples/lending/p2pk.simf) | The borrower's plain Schnorr payout address — the [Hello World](../recipes/01-hello-world-p2pk.md) program, reused for where the principal lands. |

**Four NFTs.** The protocol mints four single-unit Liquid assets at construction.
Two are bearer *auth tokens* (whoever holds it can act); two encode the loan terms
in their amount field so the offer is self-describing on-chain:

| NFT | Carries | Used for |
|-----|---------|----------|
| **Borrower NFT** | nothing (amount = 1) | proves a transaction is the borrower's; co-spent in setup and repayment |
| **Lender NFT** | nothing (amount = 1) | the lender's bearer token; needed to liquidate and to drain the vault |
| **First Parameters NFT** | interest rate, expiry, decimals — bit-packed into its amount | publishes the loan terms; checked by every covenant |
| **Second Parameters NFT** | collateral & principal base amounts — bit-packed into its amount | publishes the amounts; checked by every covenant |



## The lifecycle

The whole protocol is one `lending_contract` **class**, and the manifest closes
with an optional `lifecycle` block that names the state machine its methods walk
through:

```json
"lifecycle": {
  "states": ["nfts_issued", "offer_open", "loan_active", "repaid", "liquidated", "cancelled"],
  "entry_actions": ["IssueUtilityNFTs"],
  "transitions": {
    "IssueUtilityNFTs":           { "to": "nfts_issued" },
    "LockCollateral":             { "from": "nfts_issued", "to": "offer_open" },
    "CancelOffer":                { "from": "offer_open",  "to": "cancelled",  "unilateral": true },
    "SetupLending":               { "from": "offer_open",  "to": "loan_active" },
    "RepayLoan":                  { "from": "loan_active", "to": "repaid",     "cooperative": true },
    "LiquidateAfterExpiry":       { "from": "loan_active", "to": "liquidated", "unilateral": true },
    "ClaimPrincipalWithInterest": { "from": "repaid",      "to": "settled" }
  }
}
```

> **`lifecycle` is documentation-only.** The [spec](https://github.com/stringhandler/tx_manifest_spec/blob/main/Spec.md)
> lists it among the top-level fields as *"named states, transitions, execution
> paths"* and marks it purely informative — **nothing on-chain depends on it, and
> no tool is required to enforce it.** It exists so a reader (or a diagram
> renderer) can see the intended state machine at a glance without tracing every
> method's inputs and outputs. It may be dropped from a future revision; treat it
> as a map, not machinery.

The block has three parts:

- **`states`** — the named states an instance can be in. They're free-form labels;
  the `from`/`to` fields below refer to them. (`settled` appears as a `to` target
  without being listed — a reminder that this section is descriptive, not
  validated.)
- **`entry_actions`** — the methods that *create* a fresh instance rather than
  advancing an existing one. Here it's the constructor, `IssueUtilityNFTs`.
- **`transitions`** — one entry per method, each naming the state it moves *from*
  and *to*, plus two optional flags described below. A transition with no `from`
  (the constructor) is an entry point.

Rendered, those transitions are the protocol's flow:

```text
                    IssueUtilityNFTs
                          │   (borrower mints 4 NFTs + computes covenant hashes)
                          ▼
                    ┌───────────┐
                    │ nfts_issued│
                    └───────────┘
                          │   LockCollateral  (borrower)
                          ▼
                    ┌───────────┐   CancelOffer (borrower, unilateral)
                    │ offer_open │ ─────────────────────────────►  cancelled
                    └───────────┘
                          │   SetupLending  (lender accepts)
                          ▼
                    ┌───────────┐
                    │loan_active │
                    └───────────┘
                       │        │
        RepayLoan      │        │   LiquidateAfterExpiry
     (borrower,        │        │   (lender, unilateral,
      cooperative)     ▼        ▼    after LOAN_EXPIRATION_TIME)
                 ┌────────┐  ┌───────────┐
                 │ repaid │  │ liquidated│
                 └────────┘  └───────────┘
                       │
        ClaimPrincipalWithInterest (lender drains the vault)
                       ▼
                    settled
```

The two optional flags annotate *who* a transition needs. `"unilateral": true`
marks the two escape hatches — `CancelOffer` and `LiquidateAfterExpiry` — that one
party can take without the other's cooperation; that's the whole point of a
trustless protocol, the exits don't depend on the counterparty playing along.
`"cooperative": true` marks `RepayLoan` as the happy path both sides want. The
flags don't *do* anything on-chain — the covenants are what actually enforce who
can spend — but they tell a reader at a glance which transitions are adversarial
and which are mutual.

## How this chapter is organised

The walkthrough follows the lifecycle across four pages, each building one phase
and pulling in the recipes that introduced its pieces:

1. **[Issuing the NFTs & encoding the terms](./lending-issuance.md)** — the
   `IssueUtilityNFTs` constructor: minting four NFTs from issuance inputs,
   bit-packing the loan terms into Parameter NFT amounts, and computing the web of
   interdependent covenant hashes that every later step relies on.
2. **[Opening the offer](./lending-offer.md)** — `LockCollateral` puts the
   collateral and NFTs on-chain behind the `pre_lock` covenant, with an `op_return`
   discovery beacon and a pre-build `validations` check.
3. **[Accepting or cancelling the offer](./lending-accept.md)** — the two spending
   paths of `pre_lock`: the lender's `SetupLending` (with the `required_index`
   discipline a covenant demands) versus the borrower's `CancelOffer`.
4. **[Settling: repay, liquidate, withdraw](./lending-settlement.md)** — the
   borrower's `RepayLoan` versus the lender's `LiquidateAfterExpiry` on the
   `lending` covenant, then draining the principal vault with
   `ClaimPrincipalWithInterest`.

## Before you run it

The CLI is `tx-manifest-wallet`, aliased throughout the book to `txw`. Do the
[one-time setup](../getting-started/setup.md) first. Because this protocol has two
roles, create **two** wallets and fund both from the testnet faucet:

```sh
txw create-wallet --out borrower.json
txw create-wallet --out lender.json
# fund each from https://liquidtestnet.com/faucet, then:
txw sync --wallet borrower.json
txw sync --wallet lender.json
```

> **Where's the funding address?** Run `info` on each wallet and copy the receive
> address it prints, then paste that into the [faucet](https://liquidtestnet.com/faucet):
>
> ```sh
> txw info --wallet borrower.json   # copy the receive address, fund it, repeat for lender.json
> ```
>
> See [Fund and sync](../getting-started/setup.md#fund-and-sync) for the full walk
> through.

Get oriented with `describe` and `validate` before building anything — `describe`
prints the classes, methods, and lifecycle; `validate` checks the manifest is
internally consistent:

```sh
txw describe examples/lending/txmanifest.json
txw validate examples/lending/txmanifest.json
```

Then start with [Issuing the NFTs](./lending-issuance.md).

> **This is the most involved example in the book.** If you haven't worked
> through [Hello World](../recipes/01-hello-world-p2pk.md) and the
> [Last Will covenant](../recipes/04b-last-will.md) yet, do those first — they
> introduce the single-key and multi-path patterns this protocol composes at
> scale.
