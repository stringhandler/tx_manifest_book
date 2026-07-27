# What is a manifest?

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

**A manifest formalises that document.** The same information that used to go
into prose — *"the pre-lock UTXO must be at input index 0, the collateral goes to
output 2, the borrower's NFT must be co-spent"* — is expressed as structured JSON
that tools can read.

## The three-file model

A live contract is described by **three companion files**:

| File | Naming | What it holds | Lifetime |
|------|--------|---------------|----------|
| **Manifest** | `txmanifest.json` | The protocol definition: classes, actions, inputs, outputs, witnesses. | Static — shared by every deployment. |
| **Instance file** | `<name>.instance.json` | The compile-time parameters for *one* deployment (this borrower's pubkey, this loan's amount). | Created when the contract is instantiated. |
| **State file** | `<name>.state.json` | The live on-chain UTXO set for this instance. | Updated after every broadcast. |

The manifest is the cookbook recipe; the instance file is the specific
ingredients you bought; the state file is what's currently in the pot.

For the first several recipes we work only with the **manifest** — the other
two are introduced in [Instance, state & constructors](../recipes/10-instance-state-constructors.md).

## What a manifest contains

Everything a wallet needs to build the protocol's transactions without reading
the covenant source: the contract types and their compile-time parameters, the
on-chain states those contracts can create, the valid transactions between them,
and the state machine tying it together.

[Anatomy of a manifest](./anatomy.md) dissects each section in turn. But first,
let's get the tooling ready.
