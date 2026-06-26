# Setup

To *run* the recipes in this book (not just read them) you need the
`compose-wallet` CLI, a wallet, and a connection to a Liquid testnet Esplora
server.

> **`compose-wallet` is an example implementation of a wallet.** It is a
> reference tool that consumes compose files and walks through the full
> build-and-sign lifecycle so the recipes in this book are runnable. It is not
> the only way to consume a compose file — any wallet can implement the same
> lifecycle. If you are building your own wallet, see the
> [Wallet implementation guide](../appendix/wallet-implementation.md) for the
> execution lifecycle a wallet follows when executing an action.

## Build the CLI

The CLI lives in [`tools/compose-wallet`](https://github.com/stringhandler/s-compose/tree/main/tools/compose-wallet)
and is a standard Cargo project. From the repository root:

```sh
cargo build --manifest-path tools/compose-wallet/Cargo.toml
```

Throughout the book commands are written as `cargo run -- <subcommand>`, run from
inside `tools/compose-wallet`:

```sh
cd tools/compose-wallet
cargo run -- --help
```

> If you prefer, build a release binary and call it directly:
> `cargo build --release` then `./target/release/compose-wallet --help`.
> The `cargo run --` form is used everywhere below because it always picks up
> your latest changes.

## Configure the network and backend

The CLI keeps a small config file with two keys: the default network and the
default Esplora URL. Set them once:

```sh
cargo run -- config default_network testnet
cargo run -- config default_esplora https://blockstream.info/liquidtestnet/api
```

Run `config` with no arguments to print the current values:

```sh
cargo run -- config
```

Most subcommands also accept `--network` and `--esplora` flags to override the
defaults per-invocation.

## Create a wallet

```sh
cargo run -- create-wallet --out wallet.json
```

This writes a new HD wallet to `wallet.json`. Add `--mainnet true` for a mainnet
wallet; by default it follows your configured `default_network`.

Inspect it — fingerprint, master xpub, oracle key, and a receive address:

```sh
cargo run -- info --wallet wallet.json
```

The wallet derives keys on the BIP86 (taproot) paths the spec expects:

| Path (testnet) | Path (mainnet) | Role |
|----------------|----------------|------|
| `m/86h/1h/0h/0/0` | `m/86h/0h/0h/0/0` | Wallet signing key |
| `m/86h/1h/1h/0/0` | `m/86h/0h/1h/0/0` | Oracle key |

A `compile_params` entry with `source: { "type": "wallet_key" }` is auto-filled
from the first path; `oracle_key` from the second. (More on this in
[Parameters & validations](../recipes/02-params-and-validations.md).)

## Fund and sync

Send some Liquid testnet L-BTC to the address printed by `info`. If you don't have
any, grab some from the [Liquid testnet faucet](https://liquidtestnet.com/faucet).
Then sync the wallet against Esplora:

```sh
cargo run -- sync --wallet wallet.json
```

`sync` scans the chain, updates the persisted wallet state, and prints your
balance. To re-print the last known balance without hitting the network:

```sh
cargo run -- get-balance --wallet wallet.json
```

## Prepare UTXOs for an action

Many actions need several separate UTXOs (one per input). The `prepare`
subcommand inspects an action and, if the wallet doesn't have enough discrete
UTXOs, builds and broadcasts a split transaction to create them:

```sh
cargo run -- prepare ../../p2pk_simplicity.compose.json Pay --wallet wallet.json
```

You can also split manually:

```sh
cargo run -- split -n 4 --asset lbtc --amount-each 10000 --wallet wallet.json
```

With a funded, synced wallet you're ready for the first recipe:
[Hello World: Pay-to-Public-Key](../recipes/01-hello-world-p2pk.md).
