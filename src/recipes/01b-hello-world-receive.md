# Hello World, Part 2: Spending the output

> **Problem.** Take the covenant output you created in
> [Part 1](./01-hello-world-p2pk.md) and spend it back into your wallet — by
> producing a signature that satisfies the on-chain program.

Part 1's `Pay` action only *built* a covenant output; it locked funds into a
`p2pk_output` and recorded that UTXO in the state file. This lesson adds the
`Receive` action, which *spends* it. Three new things have to come together:

1. **The state file locates the UTXO.** We never type a txid — the tool reads
   `txmanifest.state.json` and finds the live `p2pk_output` entry.
2. **The same key rebuilds the same address.** The output is locked at an address
   derived from the recipient's pubkey. To spend it, the tool must recompile
   `p2pk.simf` with that *same* key and confirm the address matches.
3. **A witness satisfies the program.** `p2pk.simf` demands a BIP340 signature
   over the transaction. `Receive` provides one.

> **Prerequisites.** You must have run `Pay` first, so `txmanifest.state.json` holds a
> `p2pk_output`. Crucially, in Part 1 you must have locked the funds to **one of
> your own wallet's keys** (e.g. the key from `info`) — because spending now
> requires *signing* with that key's private half. If you paid to someone else's
> pubkey, only they can run `Receive`.

## The `Receive` action

Add this action alongside `Pay` in `txmanifest.json`:

```json
    "Receive": {
      "description": "Spend a p2pk output back into your wallet. Requires a BIP340 signature from the pubkey the output was locked to.",
      "params": {
        "pubkey": {
          "type": "pubkey",
          "description": "The x-only public key the output was locked to in Pay. Must be one of your own wallet's keys so the wallet can sign the spend."
        }
      },
      "inputs": [
        {
          "id": "p2pk_in",
          "description": "The p2pk covenant UTXO to spend, located via the state file by its utxo_type.",
          "utxo_source": {
            "utxo_type": "p2pk_output",
            "compile_params": { "PUB_KEY": "params.pubkey" }
          },
          "witnesses": {
            "SIGNATURE": {
              "type": "Signature",
              "sig_type": "sig_hash_all",
              "source": { "type": "wallet", "key": "params.pubkey" },
              "description": "BIP340 Schnorr signature over the whole transaction, from the recipient key."
            }
          }
        },
        {
          "id": "fee_input",
          "description": "Wallet L-BTC UTXO to pay the network fee.",
          "utxo_source": "wallet",
          "asset": "lbtc",
          "optional": true
        }
      ],
      "outputs": [
        {
          "id": "received_out",
          "description": "The reclaimed funds, sent to your wallet.",
          "destination": "wallet",
          "asset": "lbtc",
          "amount_sat": "p2pk_in.amount_sat"
        },
        {
          "id": "fee_change",
          "description": "L-BTC change from the fee input.",
          "destination": "change",
          "asset": "lbtc",
          "optional": true
        }
      ]
    }
```

## How it works

**The input comes from the state file, not your wallet.** `p2pk_in`'s
`utxo_source` is `{ "utxo_type": "p2pk_output" }`. Unlike a `"wallet"` input,
this tells the tool to look in `txmanifest.state.json` for a live UTXO of that type —
the very one `Pay` recorded. That's why this lesson "requires the state file":
without it the tool has no idea the UTXO exists.

**`compile_params` rebuilds the covenant address.** A covenant UTXO has no key in
the usual sense — its address *is* the compiled program. To spend it, the tool
recompiles `p2pk.simf` and checks the resulting Taproot address against the one
the funds are sitting at. That compile needs `PUB_KEY`, so the input carries the
same per-site map you saw on the `Pay` output:
`{ "PUB_KEY": "params.pubkey" }`. **Supply the identical pubkey you used in
`Pay`** — a different key compiles to a different address, and the UTXO simply
won't match.

**The `SIGNATURE` witness satisfies the program.** Recall `p2pk.simf`:

```rust
let sig: Signature = witness::SIGNATURE;
jet::bip_0340_verify((param::PUB_KEY, jet::sig_all_hash()), sig);
```

The program reads `witness::SIGNATURE` and verifies it against `PUB_KEY`. The
input's `witnesses` map provides exactly that name:

- `type: "Signature"` — the tool computes the signature itself rather than taking
  a literal value.
- `sig_type: "sig_hash_all"` — the message to sign is Simplicity's
  `sig_all_hash`, a commitment over the whole transaction. (This is *not* the
  classic Bitcoin/Elements `SIGHASH_ALL`; it's Simplicity's own hash. See
  [Witnesses](./04-witnesses.md).)
- `source: { "type": "wallet", "key": "params.pubkey" }` — the tool searches your
  wallet's BIP86 derivation paths for the private key matching that pubkey, signs
  the hash, and injects the 64-byte signature as the `SIGNATURE` witness.

Because the program checks the signature against the *same* `PUB_KEY` baked into
the address, only the holder of that key can produce a spend that succeeds.

**Why the separate `fee_input`.** `received_out` returns the full
`p2pk_in.amount_sat` to your wallet, so there's nothing left over for the network
fee. The optional `fee_input` pulls a small L-BTC UTXO from your wallet; the fee
is taken from its `fee_change`. (If you'd rather, drop the fee input and lower
`received_out` by the fee instead — but a separate fee input keeps the covenant
amount clean.)

**No path selector needed.** `p2pk.simf` is a single-leaf covenant with one
witness, so there's nothing to choose — `SIGNATURE` is the only witness. Richer
covenants with multiple spending paths add a selector witness; that's
[Multiple spending paths](./06-multiple-spending-paths.md).

## Run it

With a funded, synced wallet and a `p2pk_output` already in the state file from
Part 1, run:

```sh
txw run examples/p2pk/txmanifest.json Receive \
  --network testnet --wallet wallet.json
```

`run` prompts for `pubkey` (use the **same** key as in `Pay`), finds the
`p2pk_output` in the state file, rebuilds the covenant address to confirm the
match, builds the PSET, and computes the signature. Before broadcasting it runs a
**Simplicity dry-run** — actually executing the covenant program against the
spending transaction to prove the witness satisfies it — then signs, broadcasts,
and updates the state file.

> **Tip.** Add `--debug-jets` to watch `bip_0340_verify` and `sig_all_hash`
> execute during the dry-run, or `--export-pset out.json` to inspect the spend
> without broadcasting.

## The state file after spending

A successful `Receive` consumes the covenant UTXO, so the tool **removes** it from
`txmanifest.state.json`. If that was the only entry, `utxos` is now empty:

```json
{
  "last_action": "Receive",
  "utxos": []
}
```

The funds are back in your wallet as an ordinary output. The round trip is
complete: `Pay` moved L-BTC into the covenant and added a state entry; `Receive`
spent it and removed that entry.

## Try next

You've now built *and* spent a covenant — the full lifecycle of the simplest
contract. The next recipe looks more closely at the parameters and validation
rules that drive these actions:
[Parameters & validations](./02-params-and-validations.md).
