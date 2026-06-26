# Parameters & validations

> **Problem.** Reject obviously-broken transactions *before* building them, and
> understand when a value belongs in `compile_params` versus an action's `params`.

The `Pay` action from [recipe 1](./01-hello-world-p2pk.md) would happily let you
pay an output of zero satoshis. This recipe adds a validation rule to stop that,
and along the way pins down the two kinds of parameters a compose file deals with.

## Two kinds of parameters

This trips up everyone at first, so it's worth being precise:

| | `compile_params` | action `params` |
|---|---|---|
| **When fixed** | At deploy time, once. | Per transaction, every time you run the action. |
| **Baked into the script?** | Yes — they change the covenant's address. | No — they only affect this transaction. |
| **Stored in** | the instance file | nowhere; supplied at run time |
| **Example** | `PUBKEY`, `LOAN_EXPIRATION_TIME` | `amount_sat`, `CURRENT_BLOCK_HEIGHT` |

A useful test: *"if I changed this value, would the on-chain address change?"* If
yes, it's a compile param. If it only affects which inputs/outputs this particular
transaction picks, it's an action param.

### Auto-filled params with `source`

An action param (or a user-provided compile param) can declare a `source` so the
tool fills it in without prompting:

```json
"params": {
  "BORROWER_PUB_KEY": {
    "type": "pubkey",
    "description": "Borrower's signing key.",
    "source": { "type": "wallet_key" }
  }
}
```

| `source.type` | Resolves to | Derivation path (testnet / mainnet) |
|---------------|-------------|--------------------------------------|
| `"wallet_key"` | 32-byte x-only pubkey | `m/86h/1h/0h/0/0` / `m/86h/0h/0h/0/0` |
| `"oracle_key"` | 32-byte x-only pubkey | `m/86h/1h/1h/0/0` / `m/86h/0h/1h/0/0` |

If a param has no `source`, the tool prompts you for it interactively (or you
supply it via `--params`, below). In recipe 1, `PUBKEY` has no source, so `Pay`
prompts you for it. The lending protocol's `BORROWER_PUB_KEY` (above) uses
`wallet_key`, so it's filled from the wallet silently.

## Recipe

Add a `validations` array to the `Pay` action. Each rule is checked before the
PSET is built; if any rule's expression is false, the action aborts with the
rule's error message.

```json
"Pay": {
  "description": "Lock funds into a p2pk output that only PUBKEY's owner can spend.",
  "params": {
    "amount_sat": { "type": "u64", "description": "Amount in satoshis to lock." }
  },
  "inputs":  [ ... ],
  "outputs": [ ... ],

  "validations": [
    {
      "id": "amount_nonzero",
      "description": "Must lock a positive amount.",
      "rule": { "type": "arithmetic", "expr": "params.amount_sat > 0" },
      "error": { "code": "INVALID_AMOUNT", "message": "Amount must be greater than zero" }
    }
  ]
}
```

The other rule type, `utxo_exists`, guards an action that *spends* a covenant —
you'll add one when you build the spend action in a later lesson. It checks that a
UTXO of a given type exists before the action runs:

```json
"validations": [
  {
    "id": "p2pk_exists",
    "description": "A p2pk output must exist before it can be spent.",
    "rule": { "type": "utxo_exists", "utxo_type": "p2pk_output" },
    "error": { "code": "MISSING_UTXO", "message": "No p2pk UTXO found. Has Pay been run?" }
  }
]
```

## How it works

A validation rule has four parts:

| Field | Required | Purpose |
|-------|----------|---------|
| `id` | yes | Unique name for the rule (shown in errors and logs). |
| `description` | no | Human-readable intent. |
| `rule` | yes | The check itself — see the two `type`s below. |
| `error` | no | `{ "code", "message" }` surfaced when the rule fails. |

There are two rule types:

- **`arithmetic`** — the `expr` is a [formula](./07-formulas-and-derived-params.md)
  that must evaluate to `true`. Use it for amount bounds, timelock checks,
  relationships between params. `params.amount_sat > 0` is the simplest case;
  `compile_params.LOAN_EXPIRATION_TIME < params.CURRENT_BLOCK_HEIGHT` is a real
  one from the lending protocol.
- **`utxo_exists`** — names a `utxo_type` that must have at least one live entry
  in the state file. Use it as a precondition: don't try to spend something that
  was never created.

**When validations run.** All params are resolved and all inputs are selected
*before* validations execute, so a rule can reference resolved input amounts and
assets — but it runs *before* the PSET is constructed, so a failing rule costs
nothing. Any failure aborts the whole action.

> **Error codes.** The `error.code` strings can be collected into a top-level
> `errors` map (`{ "1": "Loan has not yet expired", ... }`) that documents every
> failure mode the protocol can produce. This is optional but recommended for
> protocols a wallet will surface to users.

## Run it

Validations are invisible on the happy path. To see one fire, run `Pay` and
enter `0` when prompted for `amount_sat`:

```sh
cargo run -- run-compose ../../p2pk_simplicity.compose.json Pay \
  --network testnet --wallet wallet.json
# → aborts with: INVALID_AMOUNT: Amount must be greater than zero
```

### Supplying params non-interactively

Instead of typing params at the prompt, pass a flat JSON file of string→string
values and reference it with `--params`:

```json
{ "amount_sat": "50000", "PUBKEY": "<64-hex-char x-only pubkey>" }
```

```sh
cargo run -- run-compose ../../p2pk_simplicity.compose.json Pay \
  --network testnet --wallet wallet.json --params pay-params.json
```

The CLI also auto-discovers a per-network param file when you pass `--network`
(see the lending example's `*.testnet.json` files). An explicit `--params` file
always takes precedence.

## Try next

Validations guard the *inputs*. Next we get precise about the *outputs*: the
different destinations a value can go to, and how confidentiality is decided:
[Outputs & destinations](./03-outputs-and-destinations.md).
