# Hello World: Pay-to-Public-Key

> **Problem.** Lock a Liquid output so that only the holder of one private key can
> spend it — and meet Simplicity, the on-chain language that enforces the lock.

After the no-covenant warm-up in [Splitting a UTXO](./00-splitting-utxos.md), this
is the first contract with a real **covenant**: an on-chain program that decides
whether a UTXO may be spent. It is the manifest equivalent of
`println!("Hello, world!")`.

This lesson covers only the **`Pay`** action — locking funds into the covenant.
*Spending* those funds back out (the `Receive` action) needs a signature witness
and gets its own lesson later. The full file is
[`txmanifest.json`](https://github.com/stringhandler/txmanifest-wallet/blob/main/examples/p2pk/txmanifest.json)
under `examples/p2pk/`.

## Introducing Simplicity

Until now our outputs went to ordinary wallet addresses. A covenant output is
different: its address *is* a program. On Liquid that program is written in
**SimplicityHL** — a high-level language that compiles to Simplicity — and lives
in a `.simf` file. The compiler turns it into a 32-byte commitment (its CMR),
which becomes the output's Taproot address. To spend the output you must supply a
*witness* that makes the program succeed.

A manifest does not contain the program; it **points at the `.simf` file** and
supplies its compile-time parameters. So a real contract is now **two files** that
live side by side:

```text
your-book-folder/
├── txmanifest.json     ← the manifest (references "./p2pk.simf")
└── p2pk.simf             ← the Simplicity program, compiled into the address
```

The `source` path in the manifest is resolved relative to the manifest's
own directory, so keep the `.simf` next to it.

### The program: `p2pk.simf`

Create `p2pk.simf` with exactly this content:

```rust
fn main() {
    let sig: Signature = witness::SIGNATURE;
    jet::bip_0340_verify((param::PUB_KEY, jet::sig_all_hash()), sig);
}
```

Four pieces:

- **`witness::SIGNATURE`** — a value supplied by the spender at spend time. The
  program reads it into `sig`. (We don't supply it in this lesson because `Pay`
  only *creates* the output.)
- **`param::PUB_KEY`** — a compile-time parameter baked into the program. Different
  keys compile to different programs, and therefore different addresses. We supply
  its value from the action below.
- **`jet::sig_all_hash()`** — a *jet* (a built-in Simplicity primitive) that returns
  the signature hash committing to the whole transaction.
- **`jet::bip_0340_verify((PUB_KEY, message), sig)`** — verifies that `sig` is a
  valid BIP340 Schnorr signature over that message by `PUB_KEY`. If it isn't, the
  program fails and the spend is rejected.

In plain English: *"this output may be spent only by a signature from `PUB_KEY`
over this transaction."* That is pay-to-public-key.

## The manifest

### Start with a skeleton

Create `txmanifest.json` next to `p2pk.simf`, with every top-level section
present but empty:

```json
{
  "manifest_version": "0.1.0",
  "attestation_version": "1",
  "protocol": "p2pk-simplicity",
  "description": "Hello World — Pay-to-public-key using a Simplicity checksig program on Liquid.",
  "chain": "liquid",

  "utxo_types": {},
  "actions": {}
}
```

That is the whole shape: the **envelope** (the first five fields, covered in
[Anatomy of a manifest](../getting-started/anatomy.md)) followed by two empty
data sections we'll fill in below — the UTXO type and the `Pay` action. There's
no `compile_params` block: the recipient's key is a *runtime* parameter of the
action, not a value baked in at deploy time. (And no `lifecycle`, since this
lesson has a single action.)

We'll fill the two sections in order.

### Fill in the UTXO type

The UTXO type names the on-chain state and points at the `.simf` program. Replace
the empty `utxo_types` with:

```json
  "utxo_types": {
    "p2pk_output": {
      "description": "A Liquid UTXO locked to a pubkey via the compiled p2pk.simf program.",
      "script": {
        "type": "simplicity",
        "source": "./p2pk.simf"
      },
      "asset": "lbtc",
      "confidential": false
    }
  },
```

Notice the `script` block only *names* the program — it carries no `compile_params`
map. The program still has a `PUB_KEY` parameter to fill, but we'll supply that
per-output, from a value the action receives at run time. That's the next section.

### Fill in the `Pay` action

Finally, the action itself — the value it takes, the UTXOs it consumes, what it
creates, and what must hold before it builds. The `Pay` action declares a `pubkey`
parameter and feeds it into the covenant on the output's `destination`. Replace
the empty `actions` with:

```json
  "actions": {
    "Pay": {
      "description": "Lock funds into a p2pk output that only the pubkey's owner can spend.",
      "params": {
        "pubkey": {
          "type": "pubkey",
          "description": "The x-only public key that will be able to spend this output (the recipient)."
        },
        "amount_sat": {
          "type": "u64",
          "description": "Amount in satoshis to lock in the output."
        }
      },
      "inputs": [
        {
          "id": "funding_input",
          "description": "Wallet UTXO providing the funds.",
          "utxo_source": "wallet",
          "asset": "lbtc",
          "amount_sat": { "min_amount": "params.amount_sat" }
        }
      ],
      "outputs": [
        {
          "id": "p2pk_out",
          "description": "The funded p2pk output, locked to the recipient's pubkey.",
          "destination": {
            "utxo_type": "p2pk_output",
            "compile_params": { "PUB_KEY": "params.pubkey" }
          },
          "amount_sat": "params.amount_sat",
          "asset": "lbtc"
        },
        {
          "id": "change_out",
          "description": "Change returned to the funding wallet.",
          "destination": "change",
          "asset": "lbtc",
          "optional": true
        }
      ],
      "validations": [
        {
          "id": "amount_nonzero",
          "rule": { "type": "arithmetic", "expr": "params.amount_sat > 0" },
          "error": { "code": "INVALID_AMOUNT", "message": "Amount must be greater than zero" }
        }
      ]
    }
  }
```

The key line is the output's `destination`: alongside `utxo_type` it carries a
`compile_params` map, `{ "PUB_KEY": "params.pubkey" }`, wiring the runtime `pubkey`
into the program's `param::PUB_KEY` just for this output.

With both sections filled in you have the complete file — identical to
[`txmanifest.json`](https://github.com/stringhandler/txmanifest-wallet/blob/main/examples/p2pk/txmanifest.json)
under `examples/p2pk/`.

## How it works

**The recipient's key is a runtime parameter.** `pubkey` lives under the `Pay`
action's `params` — you supply it when you run the action (e.g. from the
recipient's `info` output). It has type `pubkey` (a 32-byte x-only BIP340 key).
Nothing about it is baked into the file at deploy time; there's no
`compile_params` block at all.

**The output wires that key into the covenant.** The `p2pk_out` destination is
`{ "utxo_type": "p2pk_output", "compile_params": { "PUB_KEY": "params.pubkey" } }`.
That `compile_params` map feeds the runtime `pubkey` into the program's
`param::PUB_KEY`. The tool compiles `p2pk.simf` with that key and derives the
covenant's Taproot address — using a NUMS internal key so the only way to spend is
through the script. Because the key is baked into the *compiled program*, two
different keys give two different `p2pk_output` addresses. (We unpack that
derivation in [Covenant UTXO types](./05-covenant-utxo-types.md).)

**`Pay` creates the covenant output.** The `p2pk_out` output sends funds to the
`p2pk_output` covenant; the tool computes that covenant's address (from
`params.pubkey`, above) and locks the funds there. The `funding_input` is a plain
`"wallet"` UTXO, auto-selected via `{ "min_amount": ... }` to cover the amount, and
the remainder returns as `change`.

**No witnesses yet.** `Pay` only *builds* the locked output — it doesn't spend a
covenant — so there's nothing to satisfy and no witness to provide. The
`witness::SIGNATURE` in `p2pk.simf` only matters when you spend the output, which
is the next lesson.

## Run it

Make sure you have a funded, synced wallet ([Setup](../getting-started/setup.md)),
and that `p2pk.simf` sits next to the manifest. From the repository root:

```sh
# Optional: check the schema first.
txw validate examples/p2pk/txmanifest.json

# Make sure the wallet has a UTXO big enough for Pay.
txw prepare examples/p2pk/txmanifest.json Pay --wallet wallet.json

# Lock funds into a p2pk output. You'll be prompted for pubkey and amount_sat.
txw run examples/p2pk/txmanifest.json Pay \
  --network testnet --wallet wallet.json
```

`run` prompts for `pubkey` and `amount_sat`, compiles `p2pk.simf` to derive
the covenant address, builds the PSET, signs the wallet input, and broadcasts. The
new `p2pk_output` is recorded in the state file, ready to be spent in a later
lesson.

> **Tip.** Add `--export-pset out.json` to write the signed PSET to a file instead
> of broadcasting, or `--debug-jets` to print every Simplicity jet call.

> **Heads up for Part 2.** If you plan to follow [Part 2](./01b-hello-world-receive.md)
> and *spend* this output, lock it to **your own** wallet key — use the pubkey from
> `info` for `pubkey`. Spending requires signing with that key's private half, so
> paying to someone else's key means only they can reclaim it.

## The state file

After a successful `Pay`, the tool records the new covenant output in a **state
file** next to your manifest, auto-named `txmanifest.state.json`:

```json
{
  "last_action": "Pay",
  "utxos": [
    {
      "utxo_type": "p2pk_output",
      "utxo_id": "p2pk_out",
      "txid": "271c1afc4e6137b77874be8d9451a84e122b4bda445d512a45e61eaaddbdaab5",
      "vout": 0,
      "amount_sat": 1000,
      "asset": "144c654344aa716d6f3abcc1ca90e5641e4e2a7f633bc09fe3baf64585819a49"
    }
  ]
}
```

This is the protocol's live on-chain state: one entry per covenant UTXO the
contract currently owns. The `p2pk_out` output you just created is now tracked as
a `p2pk_output`, keyed by its `txid`/`vout` and ready to be consumed as an input
by a later action (the `Receive` spend). Each `utxo_id` matches the `id` of the
output that produced it. When a UTXO is later spent, the tool removes it here;
when an action creates new covenant outputs, it adds them.

### No instance file

A contract has up to two companion files: an **instance file** (compile-time
field values) and a **state file** (live UTXOs). This lesson produced only the
state file — there is **no `txmanifest.instance.json`**.

Why? Instance files exist to persist a `class`'s `fields`. This contract declares
no `classes` at all — and the recipient's key (the `pubkey` action parameter) is
supplied fresh at run time, never stored. With no class fields to record, there is
nothing for an instance file to hold. Instance files first appear once we
introduce classes and constructors in
[Instance, state & constructors](./10-instance-state-constructors.md).

## Try next

You now have a covenant on-chain. The next recipe digs deeper into parameters —
the `pubkey` and `amount_sat` values you just supplied — and adds runtime
validation: [Parameters & validations](./02-params-and-validations.md).
