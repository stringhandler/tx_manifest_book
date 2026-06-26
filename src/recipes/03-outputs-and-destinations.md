# Outputs & destinations

> **Problem.** Send a transaction's value to the right place — a wallet, a change
> address, a covenant, a raw script hash, or an `OP_RETURN` — and control whether
> each output is blinded.

Every action so far produced two outputs: a covenant output and a change output.
Those are only two of the destinations available. This recipe is a tour of all of
them, plus the rules for output confidentiality.

## The output descriptor

An output descriptor has these fields:

| Field | Required | Purpose |
|-------|----------|---------|
| `id` | yes | Unique name within the action. |
| `destination` | yes | Where the value goes. See below. |
| `amount_sat` | usually | Amount in satoshis — a literal or a [formula](./07-formulas-and-derived-params.md). Omit it for a `change` destination, which auto-computes the remainder. |
| `asset` | yes | Asset ID — `"lbtc"`, a 64-char hex ID, or a param reference. |
| `description` | no | Human-readable purpose. |
| `required_index` | no | Force this output to a specific transaction index. |
| `optional` | no | If `true`, the output may be omitted (e.g. zero change). Default `false`. |
| `confidential` | no | Whether to blind this output. See the rules below. |
| `data` | no | `OP_RETURN` payload — only valid with the `op_return` destination. |

## Recipe: every destination type

### Wallet and change

```json
{ "id": "to_me",     "destination": "wallet", "amount_sat": "...", "asset": "lbtc" }
{ "id": "change_out", "destination": "change", "asset": "lbtc", "optional": true }
```

`wallet` is your primary receive address; `change` is your change address. A
`change` output needs no `amount_sat` — the tool sends whatever is left after the
other outputs and fees. It is almost always `optional` too, since that remainder
can be zero.

### An address supplied at run time

```json
{ "id": "recipient_output", "destination": "params.recipient_address", "amount_sat": "params.send_amount_sat", "asset": "lbtc" }
```

The destination is a bare string referencing an action param of type `address` —
the way to pay an arbitrary recipient address that the user supplies at run time.

### A covenant UTXO type

```json
{ "id": "p2pk_out", "destination": { "utxo_type": "p2pk_output" }, "amount_sat": "params.amount_sat", "asset": "lbtc" }
```

The tool computes the named UTXO type's P2TR address from its `.simf` source and
compile params, and locks the output there. This is how a transaction *creates* the
protocol's on-chain states — exactly what `Pay` in
[recipe 1](./01-hello-world-p2pk.md) does with `p2pk_output`.

### A raw script hash from a compile param

```json
{ "id": "relocked", "destination": { "script_hash": "compile_params.PARAMETERS_NFT_OUTPUT_SCRIPT_HASH" }, "amount_sat": 1, "asset": "compile_params.FIRST_PARAMETERS_NFT_ASSET_ID" }
```

When you already hold a 32-byte covenant script hash as a derived compile param,
embed it directly as a P2TR output without recompiling. The lending protocol uses
this to re-lock NFTs under a script-auth covenant.

### `OP_RETURN`

```json
{
  "id": "indexer_op_return",
  "destination": { "type": "op_return" },
  "amount_sat": 0,
  "asset": "lbtc",
  "data": "concat(compile_params.BORROWER_PUB_KEY, compile_params.PRINCIPAL_ASSET_ID)"
}
```

An `OP_RETURN` output is provably unspendable — its value is destroyed. Two common
uses:

1. **On-chain discovery.** Publish protocol metadata (here, the borrower's pubkey
   and principal asset, 64 bytes) so an indexer can list the contract without an
   off-chain database. The `data` field is a `concat(...)` formula.
2. **Burning a token.** Spend an NFT into `OP_RETURN` to destroy it — the lending
   protocol burns auth NFTs this way to prevent reuse.

## How confidentiality is decided

On Liquid, outputs can be **blinded** (amount and asset hidden). The tool resolves
blinding in this precedence order:

1. The per-output `confidential` field, if present.
2. The top-level `confidential_outputs` field, if present.
3. The chain default: `false` for Bitcoin, `true` for Liquid/Elements.

> **Covenants and `OP_RETURN` are always unblinded**, regardless of the settings
> above. Simplicity covenants introspect explicit amounts and asset IDs with jets
> like `current_amount` and `current_asset`; they cannot read confidential
> commitments. This is a hard constraint, not a preference — a blinded covenant
> output would be unspendable.

## Forcing output order with `required_index`

Covenants that introspect the transaction often require outputs in an exact order
("collateral at output 0, principal at output 1"). Pin an output's position with
`required_index`:

```json
{ "id": "lending_collateral_out", "required_index": 0, "destination": { "utxo_type": "lending_collateral" }, ... }
{ "id": "principal_to_borrower",  "required_index": 1, "destination": "params.borrower_address", ... }
```

Positive indices are absolute (0-based). Negative indices count from the end
(`-1` is the last output). The same field exists on inputs — see
[Multiple spending paths](./06-multiple-spending-paths.md), where covenant input
ordering matters too.

## Run it

Outputs are exercised by every action; there is no standalone command. To inspect
exactly what an action will produce without broadcasting, export the PSET and
decode it:

```sh
cargo run -- run-compose ../../p2pk_simplicity.compose.json Pay \
  --network testnet --wallet wallet.json --export-pset pay.pset.json
```

The exported file lists every output with its amount, asset, and scriptPubKey, so
you can confirm the destinations resolved as you intended.

## Try next

We've now described value flowing *out*. Spending a covenant requires *witnesses*
to satisfy its Simplicity program — signatures, path selectors, and computed
values. That's the next recipe: [Witnesses](./04-witnesses.md).
