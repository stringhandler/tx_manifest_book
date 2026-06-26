# Anatomy of a compose file

Before writing any recipes, let's look at the skeleton every compose file shares.
A compose file is a single JSON document. At the top level it has an **envelope**
of metadata fields followed by the data sections.

```json
{
  "compose_version": "0.1.0",
  "attestation_version": "1",
  "protocol": "p2pk-simplicity",
  "description": "Pay-to-public-key using a Simplicity checksig program on Liquid.",
  "chain": "liquid",

  "compile_params": { ... },
  "utxo_types":     { ... },
  "classes":        { ... },
  "actions":        { ... },
  "lifecycle":      { ... }
}
```

## The envelope

| Field | Required | Purpose |
|-------|----------|---------|
| `compose_version` | yes | Version of the SimplicityCompose format itself. Current: `"0.1.0"`. |
| `protocol` | yes | Kebab-case protocol identifier, e.g. `"simplicity-lending"`. |
| `description` | yes | Free-text summary of the whole protocol. |
| `chain` | no | `"bitcoin"`, `"liquid"`/`"elements"`, or `"cross-chain"`. Defaults to `"elements"`. |
| `attestation_version` | no | Schema version for any signatures added to the document. |
| `simplicity_hl_version` | no | SimplicityHL compiler version the scripts require. |
| `source` | no | Relative path to the top-level `.simf` file. |
| `confidential_outputs` | no | File-level default for output blinding. See [Outputs & destinations](../recipes/03-outputs-and-destinations.md). |

## The data sections

A file carries up to five data sections. Two of them — the contract's
compile-time **fields** and its **methods** — can be written one of two ways:

- **Grouped into a top-level `classes` map** (the canonical model, used by the
  [lending example](../walkthrough/lending-protocol.md)). Each entry is a
  deployable contract type bundling its `fields` and `methods`.
- **Flattened** into a top-level `compile_params` block plus a top-level
  `actions` map. Simpler, and used by the early recipes in this book.

The other sections — `utxo_types` and `lifecycle` — look the same either way.
Most files won't carry every section.

### `classes` — typed contracts (fields + methods)

A `class` is a typed contract definition: one deployable contract type with its
compile-time `fields` and the `methods` (actions) that operate on it, all grouped
under a top-level `classes` map. Richer protocols use this form — the canonical
lending file is built entirely from classes.

```json
"classes": {
  "p2pk_contract": {
    "description": "Pay-to-public-key contract.",
    "fields": {
      "PUBKEY": { "type": "pubkey", "description": "Key that controls spending." }
    },
    "methods": { "Pay": { ... }, "Receive": { ... } }
  }
}
```

A file may also carry a top-level `actions` map *alongside* `classes`, for
utility actions that don't belong to a single instance (the lending file uses
this for its `Prepare` actions). Classes, fields, and methods are covered in full
in [Instance, state & constructors](../recipes/10-instance-state-constructors.md).

### Compile-time parameters — what's baked in at deploy time

Whether they live in a class's `fields` or a top-level `compile_params` block,
compile-time values parameterize the covenant scripts: a pubkey, an asset ID, a
loan amount, an expiry height. They are fixed for a deployment — change one and
you get a different script hash, and therefore a different address.

In the canonical model they are the **fields** of a `class` (above), and their
values are stored in the instance file. Simpler single-type contracts —
including the early recipes in this book — may instead declare them in a
top-level `compile_params` block. Both forms are accepted by the tooling:

```json
"compile_params": {
  "user_provided": {
    "PUBKEY": { "type": "pubkey", "description": "Key that controls spending." }
  }
}
```

Some params are **derived** rather than supplied — computed from other params or
from the outpoints of issuance inputs. See
[Formulas & derived params](../recipes/07-formulas-and-derived-params.md).

### `utxo_types` — the on-chain states

Each UTXO type is a named on-chain state with a known script (usually a Taproot
address built from a Simplicity leaf). A wallet uses these definitions to
recognise the protocol's outputs on-chain.

```json
"utxo_types": {
  "p2pk_output": {
    "description": "A Liquid UTXO locked to PUBKEY via the compiled p2pk.simf program.",
    "script": {
      "type": "simplicity",
      "source": "./p2pk.simf",
      "compile_params": { "PUB_KEY": "PUBKEY" }
    },
    "asset": "lbtc"
  }
}
```

We cover the `script` block in detail in
[Covenant UTXO types](../recipes/05-covenant-utxo-types.md).

### `actions` — the valid transactions

Each action (or *method*) is a single transaction recipe: which UTXOs to consume
(`inputs`), what to create (`outputs`), what witnesses to provide (`witnesses`),
and what must be true before building (`validations`).

```json
"actions": {
  "Pay": {
    "description": "Pay a recipient by locking funds into a p2pk output keyed to their public key.",
    "params":  { ... },
    "inputs":  [ ... ],
    "outputs": [ ... ],
    "validations": [ ... ]
  }
}
```

> **Classes vs. top-level actions.** In richer protocols, actions are grouped
> inside a `classes.<id>.methods` block — a *class* is a typed contract with
> fields and methods. For simple, single-type contracts, actions can live
> directly under top-level `actions`. Structurally a method and an action are
> identical. We start with top-level `actions` and introduce classes in
> [Instance, state & constructors](../recipes/10-instance-state-constructors.md).

### `lifecycle` — documentation of the state machine

Purely descriptive: the named states, the transitions between them, and whether
each action needs one party (`unilateral`) or both (`cooperative`). Tools render
diagrams from it, but nothing on-chain depends on it.

```json
"lifecycle": {
  "states": ["paid", "received"],
  "transitions": {
    "Pay":     { "to": "paid" },
    "Receive": { "from": "paid", "to": "received" }
  }
}
```

## The execution model, in brief

When you run an action, a tool like `compose-wallet` performs roughly these steps:

1. **Resolve parameters** — load compile params, auto-derive wallet keys, apply overrides.
2. **Resolve inputs** — find each input UTXO (from state file, wallet, or `provided_inputs`).
3. **Compute derived params** — compile `.simf` files to get covenant script hashes.
4. **Run validations** — abort if any rule is false.
5. **Construct outputs** — evaluate amount and asset formulas, resolve destinations.
6. **Build the PSET**, sign (computing Simplicity witnesses), and broadcast.
7. **Update the state file** — remove spent UTXOs, add new covenant outputs.

You don't need to memorise this yet — each recipe touches the parts it needs. The
full sequence is in [`Spec.md` §11](https://github.com/stringhandler/s-compose/blob/main/Spec.md).

With the skeleton in hand, let's write our first contract.
