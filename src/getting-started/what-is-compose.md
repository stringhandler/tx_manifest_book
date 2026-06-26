# What is a compose file?

Any multi-UTXO protocol — whether it uses Bitcoin miniscript, Tapscript, or
Liquid Simplicity — imposes a specific transaction layout. Covenants that do
transaction introspection are especially strict: input 0 must be a specific
asset, output 1 must go to a specific script hash, output 2 must carry exactly
the right amount. The on-chain program enforces this, but someone still has to
*document* what layout it expects.

Historically that documentation was a PDF, a Notion page, or a comment in the
source. It was informal and only useful to the person who wrote it. Anyone else
building a wallet integration had to reverse-engineer the expected transaction
shapes and hope the docs were current.

**A compose file formalises that document.** The same information that used to go
into prose — *"the pre-lock UTXO must be at input index 0, the collateral goes to
output 2, the borrower's NFT must be co-spent"* — is expressed as structured JSON
that tools can read.

## The three-file model

A live contract is described by **three companion files**:

| File | Naming | What it holds | Lifetime |
|------|--------|---------------|----------|
| **Compose file** | `<name>.compose.json` | The protocol definition: classes, actions, inputs, outputs, witnesses. | Static — shared by every deployment. |
| **Instance file** | `<name>.instance.json` | The compile-time parameters for *one* deployment (this borrower's pubkey, this loan's amount). | Created when the contract is instantiated. |
| **State file** | `<name>.state.json` | The live on-chain UTXO set for this instance. | Updated after every broadcast. |

The compose file is the cookbook recipe; the instance file is the specific
ingredients you bought; the state file is what's currently in the pot.

For the first several recipes we work only with the **compose file** — the other
two are introduced in [Instance, state & constructors](../recipes/10-instance-state-constructors.md).

## What a compose file contains

A compose file has a small number of top-level sections:

- **`classes`** — the contract types. Each class has typed **fields** and
  **methods**. A class's fields are the contract's *compile-time parameters* —
  the values baked into its covenant scripts (pubkeys, asset IDs, amounts, expiry
  heights). Changing a field produces a different script, and therefore a
  different on-chain address. Field *values* are fixed per deployment and stored
  in the instance file, not in the compose file.
- **`utxo_types`** — the on-chain states the protocol can create. Each has a
  known script so a wallet can recognise these outputs on-chain.
- **`actions`** (also called *methods*) — the valid transactions. Each one says
  which UTXOs it consumes, which it produces, the amount formulas, and the
  witnesses needed to satisfy the covenants. Actions live either inside a class's
  `methods` or, for simple contracts, at the top level.
- **`lifecycle`** — documentation-only: named states, the transitions between
  them, and whether each action is cooperative or unilateral.


We dissect each of these in [Anatomy of a compose file](./anatomy.md). But first,
let's get the tooling ready.
