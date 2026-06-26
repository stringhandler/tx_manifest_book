# CLI reference

The `compose-wallet` CLI ([`tools/compose-wallet`](https://github.com/stringhandler/s-compose/tree/main/tools/compose-wallet))
executes compose actions interactively. Run from inside `tools/compose-wallet`
with `cargo run -- <subcommand>`, or build a binary and call `compose-wallet`
directly.

## Commands

### `validate <compose_file>`
Statically check a compose file's schema and report obvious problems — without
touching the network, wallet, or filesystem. Catches unknown `utxo_type`
references, outputs missing a required `amount_sat`, duplicate input/output/
validation ids, malformed destinations, unknown validation rule types,
`create_instance` referencing a missing class, unreferenced UTXO types, and
lifecycle transitions that don't match any action. Exits non-zero if any errors
are found (warnings alone still exit zero).

```sh
cargo run -- validate ../../p2pk_simplicity.compose.json
```

> Future versions will add deeper checks (compiling SimplicityHL leaves,
> verifying formula references resolve, checking `canonical_cmr` values).

### `describe <compose_file>`
Explore a compose file interactively. Presents a menu of the contract's overview,
classes, and standalone actions; drill into any class to list its fields and
methods, and into any action to see its params, inputs, outputs, witnesses, and
validations — without reading the raw JSON. When stdout is not a terminal (e.g.
piped to a file or `less`), it prints a full non-interactive dump of everything
instead.

```sh
cargo run -- describe ../../example/lending/v1/lending.compose.json
```

### `run-compose <compose_file> <action_name>`
Walk through the lifecycle of a compose action interactively: resolve params and
inputs, validate, build the PSET, dry-run the Simplicity covenant, sign, and
broadcast.

| Flag | Default | Purpose |
|------|---------|---------|
| `--network <net>` | config `default_network` | Network for param-file auto-discovery. |
| `--params <file>` | — | Flat JSON `string→string` overrides (takes precedence over auto-discovered file). |
| `--wallet <file>` | `wallet.json` | Wallet for input selection and signing. |
| `--data-dir <dir>` | platform data dir | Where wallet state is persisted. |
| `--instance <file>` | `<stem>.instance.json` | Instance file (compile params locked at deploy). |
| `--state <file>` | `<stem>.state.json` | State file tracking live UTXOs. |
| `--manual-inputs` | off | Prompt for every input instead of auto-selecting. |
| `--export-pset <file>` | — | Write signed PSET/tx to a file instead of broadcasting. |
| `--debug-jets` | off | Print every Simplicity jet call during dry-runs. |

### `create-wallet`
Create a new wallet JSON file. `--out <file>` (default `wallet.json`),
`--mainnet <bool>` (defaults to config network).

### `info`
Show wallet fingerprint, master xpub, oracle pubkey, and a receive address.
`--wallet <file>`.

### `sync`
Sync wallet state against an Esplora server and print the balance.
`--wallet <file>`, `--esplora <url>`, `--data-dir <dir>`.

### `get-balance`
Print the last known balance from persisted state (no network call).
`--wallet <file>`, `--data-dir <dir>`.

### `prepare <compose_file> <action_name>`
Ensure the wallet has the UTXOs an action needs; broadcasts a split transaction if
not. `--wallet`, `--esplora`, `--data-dir`, `--split-amount <sats>` (default
`10000`).

### `split`
Split a wallet asset into N equal UTXOs and broadcast. `-n/--count <N>`,
`--asset <hex|lbtc>` (default `lbtc`), `--amount-each <sats>` (optional — splits
balance evenly if omitted), `--wallet`, `--esplora`, `--data-dir`.

### `config [key] [value]`
With no args, print config. With `key value`, set it. Valid keys:
`default_network` (`testnet`|`mainnet`), `default_esplora` (URL).

## Typical session

```sh
cd tools/compose-wallet
cargo run -- config default_network testnet
cargo run -- config default_esplora https://blockstream.info/liquidtestnet/api
cargo run -- create-wallet --out wallet.json
cargo run -- info --wallet wallet.json          # → fund this address
cargo run -- sync --wallet wallet.json
cargo run -- prepare ../../p2pk_simplicity.compose.json Pay --wallet wallet.json
cargo run -- run-compose ../../p2pk_simplicity.compose.json Pay --network testnet --wallet wallet.json
```
