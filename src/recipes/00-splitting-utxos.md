# Splitting a UTXO

> **Problem.** Turn one large wallet UTXO into several smaller ones — a handy
> warm-up, and a common prerequisite for actions that need several discrete input
> UTXOs.

Before the first *real* contract, here is the gentlest possible compose file: no
covenants, no witnesses, no compile parameters. Just one input and a handful of
outputs, all to your own wallet. It does one useful thing — split a UTXO into four
equal pieces — and in doing so introduces the bare skeleton every compose file
shares.

The full file is
[`utxo_split.compose.json`](https://github.com/stringhandler/s-compose/blob/main/utxo_split.compose.json)
in the repository root.

## Recipe

```json
{
  "compose_version": "0.1.0",
  "attestation_version": "1",
  "protocol": "utxo-split",
  "description": "Split one wallet UTXO into four equal wallet UTXOs.",
  "chain": "liquid",

  "actions": {
    "Split": {
      "description": "Split a wallet UTXO into four outputs of amount_each, returning any remainder (less fees) as change.",

      "params": {
        "amount_each": {
          "type": "u64",
          "description": "Satoshis to place in each of the four output UTXOs."
        }
      },

      "inputs": [
        {
          "id": "funding_input",
          "description": "A wallet UTXO large enough to cover four outputs plus fees.",
          "utxo_source": "wallet",
          "asset": "lbtc",
          "amount_sat": { "min_amount": "params.amount_each * 4" }
        }
      ],

      "outputs": [
        { "id": "split_0", "destination": "wallet", "amount_sat": "params.amount_each", "asset": "lbtc" },
        { "id": "split_1", "destination": "wallet", "amount_sat": "params.amount_each", "asset": "lbtc" },
        { "id": "split_2", "destination": "wallet", "amount_sat": "params.amount_each", "asset": "lbtc" },
        { "id": "split_3", "destination": "wallet", "amount_sat": "params.amount_each", "asset": "lbtc" },
        { "id": "change_out", "destination": "change", "asset": "lbtc", "optional": true }
      ],

      "validations": [
        {
          "id": "amount_nonzero",
          "rule": { "type": "arithmetic", "expr": "params.amount_each > 0" },
          "error": { "code": "INVALID_AMOUNT", "message": "amount_each must be greater than zero" }
        }
      ]
    }
  }
}
```

## How it works

**The whole envelope, and nothing else.** This file has the required top-level
fields (`compose_version`, `protocol`, `description`, `chain`) and a single
`actions` block. There are no `utxo_types` (no on-chain covenant states), no
`compile_params` (nothing is baked into a script), and no `classes`. A compose
file can be this small.

**One action parameter.** `amount_each` is an action `param` of type `u64` — you
supply it each time you run `Split`. It is *not* a compile param: it doesn't change
any script or address, it only affects the amounts in this one transaction.

**One wallet input.** `funding_input` draws from `utxo_source: "wallet"` — any
wallet-controlled UTXO. Its `amount_sat` uses the `{ "min_amount": ... }` form
with a formula, `params.amount_each * 4`, so the tool auto-selects a UTXO worth at
least four shares. (Operators like `*` are covered in
[Formulas & derived params](./07-formulas-and-derived-params.md).)

**Four equal outputs, plus change.** Each `split_n` output sends `amount_each` to
`destination: "wallet"`, your own receive address. The final
`change_out` has no `amount_sat` — a `change` destination automatically receives
whatever is left after the four outputs and the fee, and it's `optional` so the
action still works if that remainder is zero.

**No witnesses.** Every input here is an ordinary wallet UTXO, which the wallet
library signs with a standard Schnorr signature. Witnesses only appear when you
spend a *covenant* — which is exactly what the next recipe introduces.

## Run it

Make sure you have a funded, synced wallet ([Setup](../getting-started/setup.md)).
From `tools/compose-wallet`:

```sh
# Optional: check the compose file's schema before running anything.
cargo run -- validate ../../utxo_split.compose.json

cargo run -- run-compose ../../utxo_split.compose.json Split \
  --network testnet --wallet wallet.json
```

You'll be prompted for `amount_each`; the tool selects an input, builds the four
outputs plus change, signs, and broadcasts. Afterwards your wallet holds four
fresh UTXOs.

> **The built-in shortcut.** Because splitting is so common, the CLI ships it as a
> first-class command — no compose file needed:
>
> ```sh
> cargo run -- split -n 4 --asset lbtc --amount-each 10000 --wallet wallet.json
> ```
>
> And `prepare` will split automatically when an action needs more UTXOs than the
> wallet currently has:
>
> ```sh
> cargo run -- prepare ../../p2pk_simplicity.compose.json Pay --wallet wallet.json
> ```

## Try next

That's the skeleton. The next recipe adds the first real Simplicity covenant — a
UTXO type, a script, and the witnesses to spend it:
[Hello World: Pay-to-Public-Key](./01-hello-world-p2pk.md).
