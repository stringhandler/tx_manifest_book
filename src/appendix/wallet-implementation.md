# Wallet implementation guide

**Audience:** Wallet implementors consuming compose files to build and sign
transactions.

The [`compose-wallet`](./cli-reference.md) CLI used throughout this book is an
*example* implementation of the lifecycle described here. Any wallet can consume
a compose file by following the same steps. This page describes the execution
lifecycle a wallet follows when executing an action from a compose file. Field
definitions are not duplicated here; refer to
[`Spec.md`](https://github.com/stringhandler/s-compose/blob/main/Spec.md) for the
authoritative field reference.

---

## Lifecycle

The following steps are executed in order for each action execution.

### 1. Parse

Read `compile_params.user_provided` and the target action's `params` and `args`.
Determine which values the user must supply upfront. Values already fixed by
`provided_inputs.params` are excluded from user prompting.

### 2. User inputs args and params

Prompt the user for all required `args` and `params` values not already covered
by `provided_inputs`. Present `description` fields as guidance text.

### 3. Input selection

For each input in the action's `inputs` array, attempt to auto-select a UTXO
satisfying the input's `utxo_source`, `asset`, and `amount_sat` constraints.
Inputs already fixed by `provided_inputs.inputs` are used verbatim — do not
prompt for these.

- For ambiguous cases (multiple candidates) or when auto-select is disabled by
  wallet policy, prompt the user to choose.
- User may opt into auto-select depending on wallet implementation.
- Validate each `provided_inputs` UTXO against chain state: confirm it is unspent
  and its `script_pubkey` matches the expected script.

### 4. `on_input_resolved` hooks run

Execute all hooks declared in `hooks.on_input_resolved`, in declaration order
(the order they appear in the file). This is the only ordering guarantee.

Each hook is keyed by input id and runs a SimplicityHL program that sets one or
more `compile_params.DERIVED_PARAM` values. The execution context available to
each hook:

- Resolved input outpoints (txid, vout), amounts, and assets for all inputs
  resolved so far.
- All `compile_params` set to date, including values set by earlier hooks in the
  same pass.

All hooks must complete before any validation runs. Subsequent validations and
output formulas may depend on the derived params set here.

**On-chain context jets** (`current_index`, `input_script_hash`, etc.) are
**not** available at build time. Those jets execute only during on-chain script
evaluation, not during transaction construction.

### 5. Outputs constructed

Build the transaction outputs from the action's `outputs` array. Evaluate each
`amount_sat` formula using the now-complete `compile_params` context
(user-provided plus all hook-derived values), resolved input amounts, and action
`args`/`params`. Resolve output `destination` fields to concrete scriptPubKeys.

### 6. Fee rate chosen and applied

Estimate the transaction fee or prompt the user for a fee rate. Apply the fee to
the transaction, adjusting any `"change"` output accordingly.

### 7. `on_validate` hook runs (if present)

If the action declares an `on_validate` hook, run the full SimplicityHL program
against the current transaction state. The program returns `Option<u16>`:

- `None` — validation passes; continue.
- `Some(n)` — validation fails with error code `n`; look up `n` in the top-level
  `errors` map and surface the description to the user. Flow returns to step 3.

### 8. 1-liner validations run

Execute each entry in the action's `validations` array, in declaration order.
Each validation evaluates its `rule` against the current transaction state.

- A failing `arithmetic` or `simplicity_hl` validation produces an error code
  from the entry's `error.code` field.
- A failing `utxo_exists` validation produces the same.

### 9. On any validation error

Look up the error code (string key) in the top-level `errors` map to obtain the
English-language description. Surface this to the user. The user adjusts their
inputs or params and flow returns to step 3.

### 10. Fee review / adjustment → PSET created

Present the user with the final fee amount. If the user adjusts the fee rate,
rerun from step 6. Signatures are not yet present at this point, so there is no
witness-invalidation problem.

Once the user confirms, construct the PSET. **This is the boundary between
compose-level reasoning and standard Elements/Bitcoin wallet machinery.** A
wallet that does not implement SimplicityCompose can receive the PSET from this
point onwards and handle signing and broadcast normally.

### 11. Wallet signs

Populate witnesses into the PSET per the action's `witnesses` map. For each
witness descriptor, produce the required data (signatures, preimages,
SimplicityHL-typed values, etc.) as specified by the `source` type. Pre-computed
witnesses from `provided_inputs.witnesses` are included verbatim.

### 12. Simplicity dry-run

Execute the covenant scripts on all inputs against the signed PSET. This is a
local simulation of on-chain script execution; it does not broadcast.

A dry-run failure indicates a bug in the compose file or wallet implementation,
**not a user error**. Surface it as an internal error with the relevant input
index and script. Do not ask the user to retry.

This step is distinct from the compose validations in steps 7–8. Compose
validations are pre-flight business-logic checks expressible without a full
Simplicity interpreter. The dry-run is the final cryptographic and covenantal
correctness check, confirming that the on-chain scripts will accept the
constructed transaction.

### 13. Broadcast

Finalise and extract the transaction from the PSET. Broadcast to the network, or
hand off to an external broadcast service.

---

## Execution context for SimplicityHL code

The following are available to all SimplicityHL code at build time (hooks and
validations):

| Available | Description |
|---|---|
| Resolved input outpoints | txid and vout for each resolved input |
| Resolved input amounts and assets | `amount_sat` and `asset` for each resolved input |
| `compile_params` | All user-provided values plus any values set by hooks that have already run |
| Action `args` and `params` | Runtime values supplied by the user |

The following are **not** available at build time:

| Not available | Reason |
|---|---|
| `current_index`, `input_script_hash`, and other introspection jets | These are on-chain execution context — they only exist when a Simplicity program runs inside the node during transaction validation, not during wallet-side transaction construction. |

---

## Error codes

Error codes are `u16` values. The compose file's top-level `errors` field maps
numeric codes to English-language descriptions:

```json
"errors": {
  "1001": "Collateral amount is below the minimum required for this loan.",
  "1002": "Loan has not yet expired; liquidation is not permitted."
}
```

Both `on_validate` (step 7) and per-entry `validations` (step 8) produce error
codes. The wallet looks up the code in `errors` and displays the description to
the user.

**Localisation.** Other locales are provided as separate JSON files sharing the
same numeric keys — the compose file itself carries only the English
descriptions. Wallet implementations that support multiple locales load the
appropriate locale file and index into it by the same code.

---

## Notes on `provided_inputs`

When a compose file arrives with a `provided_inputs` section (e.g. from a DEX
front-end or counterparty):

- Treat every entry in `provided_inputs.inputs` as fixed — do not prompt the user
  to select these UTXOs.
- Treat every entry in `provided_inputs.params` as fixed — do not prompt the user
  for these values.
- Include every entry in `provided_inputs.witnesses` verbatim in the PSET — do
  not re-derive or overwrite.
- Validate all provided UTXOs against chain state before proceeding (step 3).
- Validate pre-computed witnesses cryptographically before including them (step
  11).

`provided_inputs` data arrives from an untrusted source. See
[`Spec.md`](https://github.com/stringhandler/s-compose/blob/main/Spec.md) Section
17 for the full security requirements.
