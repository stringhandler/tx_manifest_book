# Setup

To *run* the recipes in this book (not just read them) you need the
`tx-manifest-wallet` CLI, a wallet, and a connection to a Liquid testnet Esplora
server.

> **`tx-manifest-wallet` is an example implementation of a wallet.** It is a
> reference tool that consumes manifests and walks through the full
> build-and-sign lifecycle so the recipes in this book are runnable. It is not
> the only way to consume a manifest — any wallet can implement the same
> lifecycle. If you are building your own wallet, see the
> [Wallet implementation guide](../appendix/wallet-implementation.md) for the
> execution lifecycle a wallet follows when executing an action.

## Get the CLI

The wallet binary is `tx-manifest-wallet`. There are four ways to get it; pick
whichever suits you. **Option 1 (the codespace) is the quickest way to try the
recipes** — nothing to install.

> **This book aliases the binary to `txw`** purely to keep the commands short and
> readable. Every command below is written as `txw <subcommand>` — read that as
> `tx-manifest-wallet <subcommand>` if you prefer the full name. The Blockstream
> codespace ships the `txw` alias already; with the other options, add it yourself:
>
> ```sh
> alias txw=tx-manifest-wallet
> ```

### Option 1 — Blockstream Simplicity codespace (no install)

The [Blockstream Simplicity codespace](https://github.com/Blockstream/simplicity-codespace)
comes with the example wallet preinstalled and already aliased to `txw`, alongside
the SimplicityHL toolchain. Open it in GitHub Codespaces and you can run the
recipes immediately — no local setup:

```sh
txw --help
```

### Option 2 — download a release binary

If you don't want to compile, grab a prebuilt binary from the
[releases page](https://github.com/stringhandler/txmanifest-wallet/releases).
Builds are published for Linux (x86_64), macOS (Apple Silicon), and Windows
(x86_64). Download the archive for your platform, unpack it, and put
`tx-manifest-wallet` on your `PATH`:

```sh
# Example: Linux x86_64, release v0.1.0 (substitute the current version)
curl -LO https://github.com/stringhandler/txmanifest-wallet/releases/download/v0.1.0/tx-manifest-wallet-v0.1.0-x86_64-unknown-linux-gnu.tar.gz
tar xzf tx-manifest-wallet-v0.1.0-x86_64-unknown-linux-gnu.tar.gz
sudo mv tx-manifest-wallet /usr/local/bin/
txw --help
```

> Asset names are version-stamped. The macOS Apple Silicon and Windows builds are
> `tx-manifest-wallet-<version>-aarch64-apple-darwin.tar.gz` and
> `tx-manifest-wallet-<version>-x86_64-pc-windows-msvc.zip`. Check the releases
> page for the exact file name of the latest version.

### Option 3 — asdf

The [`asdf`](https://asdf-vm.com) plugin installs prebuilt release binaries (Linux
x86_64 and macOS Apple Silicon; asdf is shell-based, so Windows isn't supported):

```sh
asdf plugin add tx-manifest-wallet https://github.com/stringhandler/asdf-tx-manifest-wallet.git
asdf install tx-manifest-wallet latest
asdf set -u tx-manifest-wallet latest
txw --help
```

### Option 4 — build from source

The CLI is the [`txmanifest_wallet`](https://github.com/stringhandler/txmanifest-wallet/tree/main/txmanifest_wallet)
crate of a standard Cargo workspace. From a clone of the repository:

```sh
cargo build --release          # binary at ./target/release/tx-manifest-wallet
alias txw="$(pwd)/target/release/tx-manifest-wallet"
txw --help
```

Throughout the book, commands are written as `txw <subcommand>`. Manifest paths
like `examples/p2pk/txmanifest.json` are relative to your current directory — run
from a clone of the repository (or the codespace) to use the bundled examples.

## Configure the network and backend

The CLI keeps a small config file with two keys: the default network and the
default Esplora URL. Set them once:

```sh
txw config default_network testnet
txw config default_esplora https://blockstream.info/liquidtestnet/api
```

Run `config` with no arguments to print the current values:

```sh
txw config
```

Most subcommands also accept `--network` and `--esplora` flags to override the
defaults per-invocation.

## Create a wallet

```sh
txw create-wallet --out wallet.json
```

This writes a new HD wallet to `wallet.json`. Add `--mainnet true` for a mainnet
wallet; by default it follows your configured `default_network`.

Inspect it — fingerprint, master xpub, oracle key, and a receive address:

```sh
txw info --wallet wallet.json
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

Your new wallet is empty. Fund it with Liquid testnet L-BTC from the faucet:

1. **Get your receive address.** Run `info` and copy the receive address it prints:

   ```sh
   txw info --wallet wallet.json
   ```

   Among the output (fingerprint, xpub, oracle key) is a **receive address** — copy
   that value.

2. **Request coins from the faucet.** Open the
   [Liquid testnet faucet](https://liquidtestnet.com/faucet), paste your receive
   address into the address field, and request the funds. The faucet broadcasts a
   small amount of testnet L-BTC to your wallet.

3. **Sync the wallet** once the faucet transaction has been broadcast, so the CLI
   picks up the new UTXO from Esplora:

   ```sh
   txw sync --wallet wallet.json
   ```

`sync` scans the chain, updates the persisted wallet state, and prints your
balance. To re-print the last known balance without hitting the network:

```sh
txw get-balance --wallet wallet.json
```

## Prepare UTXOs for an action

Many actions need several separate UTXOs (one per input). The `prepare`
subcommand inspects an action and, if the wallet doesn't have enough discrete
UTXOs, builds and broadcasts a split transaction to create them:

```sh
txw prepare examples/p2pk/txmanifest.json Pay --wallet wallet.json
```

You can also split manually:

```sh
txw split -n 4 --asset lbtc --amount-each 10000 --wallet wallet.json
```

With a funded, synced wallet you're ready for the first recipe:
[Hello World: Pay-to-Public-Key](../recipes/01-hello-world-p2pk.md).
