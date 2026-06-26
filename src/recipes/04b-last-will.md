# Worked example: a Last Will covenant

> **Problem.** Lock funds so that the owner can move them with a hot key, escape
> the arrangement with a cold key, and — if the owner goes silent for 180 days —
> let an heir claim them. All three rules enforced on-chain.

This recipe puts the last few lessons to work on a real, non-trivial contract:
the **Last Will**, adapted from the
[SimplicityHL examples](https://github.com/BlockstreamResearch/SimplicityHL). It
has **three spending paths**, a **relative timelock**, and a **recursive
covenant** — and it's a chance to see a multi-path
[witness](./04-witnesses.md) selector in a complete file.

The three paths:

| Path | Who | When | Effect |
|------|-----|------|--------|
| **Refresh** | owner's *hot* key | any time | moves the funds but **repeats the covenant** |
| **ColdBreak** | owner's *cold* key | any time | spends out, **ending** the covenant |
| **Inherit** | heir's key | after **180 days** of no movement | spends out to the heir |

The cold key is the escape hatch; the hot key is for everyday moves and is forced
to re-lock; the inheritor is the dead-man's switch.

## The program

The contract lives in
[`last_will.simf`](https://github.com/stringhandler/txmanifest-wallet/blob/main/examples/last_will/last_will.simf).
Two adaptations from the upstream example make it work with tx-manifest-wallet:

1. **The keys and the timelock are compile parameters** (`param::INHERITOR_PUB_KEY`,
   `param::HOT_PUB_KEY`, `param::COLD_PUB_KEY`, and `param::INHERIT_BLOCKS`) instead
   of hardcoded constants, so the manifest can wire them — exactly like
   `PUB_KEY` in Hello World.
2. **The path is chosen by a dedicated `SPEND_PATH` witness, and each signature
   is its own witness.** The upstream version nested the signatures *inside* the
   selector; tx-manifest-wallet computes signatures as standalone `Signature`
   witnesses, so we split them out (the idiom from
   [Witnesses](./04-witnesses.md)).

```rust
fn main() {
    match witness::SPEND_PATH {
        Left(inherit: ()) => inherit_spend(witness::INHERITOR_SIG),
        Right(cold_or_hot: Either<(), ()>) => match cold_or_hot {
            Left(cold: ()) => cold_spend(witness::COLD_SIG),
            Right(hot: ()) => refresh_spend(witness::HOT_SIG),
        },
    }
}
```

`SPEND_PATH` has type `Either<(), Either<(), ()>>`, so the three branches are
selected by `Left(())`, `Right(Left(()))`, and `Right(Right(()))`. Whichever
branch you take reads exactly one signature witness; the other two live on pruned
branches and are [auto-zeroed](./04-witnesses.md#the-witnesses-you-dont-supply).

The two interesting helpers:

```rust
fn inherit_spend(inheritor_sig: Signature) {
    let blocks: Distance = param::INHERIT_BLOCKS;   // configurable timelock (a compile param)
    jet::check_lock_distance(blocks);
    checksig(param::INHERITOR_PUB_KEY, inheritor_sig);
}

fn recursive_covenant() {
    assert!(jet::eq_32(jet::num_outputs(), 2));                 // exactly 2 outputs
    let this_script_hash: u256 = jet::current_script_hash();
    let output_script_hash: u256 = unwrap(jet::output_script_hash(0));
    assert!(jet::eq_256(this_script_hash, output_script_hash)); // output 0 = same covenant
    assert!(unwrap(jet::output_is_fee(1)));                     // output 1 = fee
}
```

`inherit_spend` enforces a **relative** timelock — the heir's spend is only valid
once the UTXO is `INHERIT_BLOCKS` blocks old (a compile param; ~180 days ≈ 25,920
one-minute Liquid blocks). `recursive_covenant` (used by the hot-key refresh)
forces the spend to recreate the *same* covenant in output 0 and have the explicit
fee in output 1, with nothing else.

## The manifest

A will is something you *deploy once* and then operate — exactly what a
[**class**](../getting-started/anatomy.md) models. We define a
`last_will_contract` class whose **fields** are the three keys, and whose
**methods** are the four actions. A **constructor** method (`Fund`) records the
keys in an instance file the first time you set the will up; the spend methods
read them back.

```json
"classes": {
  "last_will_contract": {
    "fields": {
      "INHERITOR_PUB_KEY": { "type": "pubkey" },
      "HOT_PUB_KEY": { "type": "pubkey" },
      "COLD_PUB_KEY": { "type": "pubkey" },
      "INHERIT_BLOCKS": { "type": "u16" }
    },
    "methods": { "Fund": { ... }, "ColdBreak": { ... }, "Refresh": { ... }, "Inherit": { ... } }
  }
}
```

The `last_will` UTXO type (top-level, as before) wires those three fields into the
program — the field names double as the compile params the script consumes:

```json
"utxo_types": {
  "last_will": {
    "script": {
      "type": "simplicity",
      "source": "./last_will.simf",
      "compile_params": {
        "INHERITOR_PUB_KEY": "INHERITOR_PUB_KEY",
        "HOT_PUB_KEY": "HOT_PUB_KEY",
        "COLD_PUB_KEY": "COLD_PUB_KEY",
        "INHERIT_BLOCKS": "INHERIT_BLOCKS"
      }
    },
    "asset": "lbtc"
  }
}
```

### The constructor: `Fund`

`Fund` does double duty — it locks the funds *and* writes the instance file. It
takes the three keys as params (two auto-filled from your wallet) and an amount,
locks a wallet UTXO into the covenant, then `create_instance` records the keys:

```json
"Fund": {
  "is_constructor": true,
    "params": {
      "INHERITOR_PUB_KEY": {
          "type": "pubkey",
          "description": "The heir's x-only public key. They can claim the funds 180 days after the last move."
      },
      "HOT_PUB_KEY": {
          "type": "pubkey",
          "description": "Owner's hot key. Auto-filled from your wallet signing key."
      },
      "COLD_PUB_KEY": {
          "type": "pubkey",
          "description": "Owner's cold key. Your wallet's oracle key — the covenant escape hatch."
      },
      "INHERIT_BLOCKS": {
          "type": "u16",
          "default": "25920",
          "description": "Blocks of inactivity before the heir may claim. ~180 days ≈ 25920 (1-minute Liquid blocks). Max 65535."
      },
      "amount_sat": {
          "type": "u64",
          "description": "Amount in satoshis to place under the will."
      }
  },
  "inputs": [
        {
            "id": "funding_input",
            "description": "Wallet UTXO providing the funds.",
            "utxo_source": "wallet",
            "asset": "lbtc",
            "amount_sat": {
                "min_amount": "params.amount_sat"
            }
        }
    ],
 "outputs": [
      {
          "id": "will_out",
          "description": "The funded last-will output.",
          "destination": {
              "utxo_type": "last_will"
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
  "create_instance": {
    "class": "last_will_contract",
    "fields": {
      "INHERITOR_PUB_KEY": "$params.INHERITOR_PUB_KEY",
      "HOT_PUB_KEY": "$params.HOT_PUB_KEY",
      "COLD_PUB_KEY": "$params.COLD_PUB_KEY",
      "INHERIT_BLOCKS": "$params.INHERIT_BLOCKS"
    }
  }
}
```

`HOT_PUB_KEY` auto-fills from your wallet signing key; `COLD_PUB_KEY` is your
wallet's oracle key (take it from `info` — see [Setup](../getting-started/setup.md));
the heir gives you `INHERITOR_PUB_KEY`. Each param value is written into the
compile params, so `will_out`'s covenant address is computed from the keys you
just supplied — *before* the instance exists. After broadcast, `create_instance`
writes those same three keys into `txmanifest.instance.json`.

> **One instance per will.** Unlike Hello World — which had no `classes` and so
> [no instance file](./01-hello-world-p2pk.md#no-instance-file) — the keys here
> are a class's fields, persisted at construction. Every later spend reads them
> from the instance, so you only enter the keys once. This is the full
> class / instance model from
> [Instance, state & constructors](./10-instance-state-constructors.md).

### Each spend reads the instance

Every spend is a method whose input is the `last_will` UTXO (found in the state
file) with a `SPEND_PATH` selector and the matching `Signature`. The signature
keys reference `compile_params.*` — the fields loaded back from the instance file,
so you never re-enter them. **ColdBreak**:

```json
"ColdBreak": {
  "inputs": [
    {
      "id": "will_in",
      "utxo_source": { "utxo_type": "last_will" },
      "witnesses": {
        "SPEND_PATH": { "type": "simplicityhl", "value": "Right(Left(()))" },
        "COLD_SIG": {
          "type": "Signature",
          "sig_type": "sig_hash_all",
          "source": { "type": "wallet", "key": "compile_params.COLD_PUB_KEY" }
        }
      }
    },
    { "id": "fee_input", "utxo_source": "wallet", "asset": "lbtc", "optional": true }
  ],

```json
"ColdBreak": {
  "inputs": [
    {
      "id": "will_in",
      "utxo_source": { "utxo_type": "last_will" },
      "witnesses": {
        "SPEND_PATH": { "type": "simplicityhl", "value": "Right(Left(()))" },
        "COLD_SIG": {
          "type": "Signature",
          "sig_type": "sig_hash_all",
          "source": { "type": "wallet", "key": "compile_params.COLD_PUB_KEY" }
        }
      }
    },
    { "id": "fee_input", "utxo_source": "wallet", "asset": "lbtc", "optional": true }
  ],
  "outputs": [
    { "id": "to_wallet", "destination": "wallet", "asset": "lbtc", "amount_sat": "will_in.amount_sat" },
    { "id": "fee_change", "destination": "change", "asset": "lbtc", "optional": true }
  ]
}
```

This is exactly the [Hello World spend](./01b-hello-world-receive.md) plus a
`SPEND_PATH` selector. **Inherit** is identical but with `SPEND_PATH = Left(())`
and `INHERITOR_SIG`.

**Refresh** is the one that's different — the covenant forces it to re-lock:

```json
"Refresh": {
  "inputs": [
    {
      "id": "will_in",
      "utxo_source": { "utxo_type": "last_will" },
      "witnesses": {
        "SPEND_PATH": { "type": "simplicityhl", "value": "Right(Right(()))" },
        "HOT_SIG": { "type": "Signature", "sig_type": "sig_hash_all", "source": { "type": "wallet", "key": "compile_params.HOT_PUB_KEY" } }
      }
    }
  ],
  "outputs": [
    {
      "id": "will_again",
      "destination": { "utxo_type": "last_will" },
      "asset": "lbtc",
      "amount_sat": "will_in.amount_sat - fee",
      "required_index": 0
    }
  ]
}
```

Three things the covenant dictates here:

- **`required_index: 0`** — `recursive_covenant` checks `output_script_hash(0)`,
  so the re-locked output *must* be output 0.
- **No change output.** The program asserts exactly two outputs (covenant + fee).
  Because this action declares **no `"change"` output**, the builder never adds
  one — it folds the L-BTC surplus into the fee. There's no separate fee input
  either: a recursive covenant can't add a wallet change output to return the
  leftover, so the will pays its own fee and shrinks by it each refresh.
- **`amount_sat: "will_in.amount_sat - fee"`** — the `fee` keyword. The will is
  re-locked at its current value *minus the network fee*. The fee output (output
  1) then ends up being exactly `fee`.

> **The `fee` keyword.** `fee` is a reserved formula word for the estimated
> network fee. It evaluates to `0` while the outputs are first assembled, then the
> tool estimates the fee from the transaction's size and re-evaluates any amount
> that used `fee` — so `will_in.amount_sat - fee` lands on the right value before
> signing. No `fee_sat` param to guess at.

> **How the builder decides on change.** A change output is added only when an
> action lists a `"destination": "change"` output. Methods that omit it — like
> `Refresh` — get exactly their declared outputs plus the fee, which is what a
> recursive covenant needs.

## Skip the prompts: a params file

`Fund` needs three pubkeys. Typing them in is error-prone, so the CLI can read
them from a **params file**: a flat JSON object of `param → value` that pre-fills
(or fully supplies) the prompts.

You don't even pass a flag. The tool **auto-discovers** a file named
`<stem>.<network>.json` next to the manifest (the `<stem>` is the manifest's
filename stem) — so for `testnet` it loads `txmanifest.testnet.json` from
`examples/last_will/`:

```json
{
  "INHERITOR_PUB_KEY": "…heir's wallet pubkey…",
  "HOT_PUB_KEY":       "…owner's wallet pubkey…",
  "COLD_PUB_KEY":      "…owner's oracle pubkey…",
  "INHERIT_BLOCKS":    "25920",
  "amount_sat":        "100000"
}
```

(An explicit `--params <file>` works too, and overrides the auto-discovered one.)

In this example the **heir is a second wallet**, so two of the keys come from one
wallet and one from another. Rather than copy three pubkeys out of `info` by hand,
the book ships scripts that do it for you:

```powershell
.\create_wallet.ps1          # owner wallet -> wallet.json
.\create_inherit_wallet.ps1  # heir wallet  -> wallet-inherit.json
.\make_params.ps1            # reads both, writes examples/last_will/txmanifest.testnet.json
```

`make_params.ps1` runs `info` on each wallet, pulls out the signing and oracle
pubkeys, and writes the params file — `INHERITOR_PUB_KEY` from the heir wallet,
`HOT_PUB_KEY` / `COLD_PUB_KEY` from the owner wallet. (`HOT_PUB_KEY` also
auto-fills from the wallet at run time, since it's a `wallet_key` source; the file
just makes every value explicit.)

## Run it

Run everything from the repository root, where the scripts put the wallets and
params file. With the params file in place, **construct** the will — `Fund` reads
every value from the file, so there's nothing to type. It locks the funds and
writes `examples/last_will/txmanifest.instance.json`:

```sh
txw run examples/last_will/txmanifest.json Fund \
  --network testnet --wallet wallet.json
```

Then the cold-key break-out is the most straightforward spend to run, since your
oracle key signs it — and you don't re-enter any keys, because they're read from
the instance:

```sh
txw run examples/last_will/txmanifest.json ColdBreak \
  --network testnet --wallet wallet.json
```

`ColdBreak` finds the `last_will` UTXO in the state file, rebuilds the covenant
address from the instance's three fields, signs with the cold (oracle) key,
dry-runs the program down the `Right(Left(()))` branch, and broadcasts.

> **Caveats on the other two paths.** `Refresh` and `Inherit` exercise covenant
> features the reference tool doesn't fully drive yet:
>
> - **`Inherit`** needs the input's **relative timelock** (nSequence) set to ≥180
>   days for `check_lock_distance` to pass, and is signed by the *heir's* key —
>   not your wallet. It's the dead-man's-switch path; treat it as illustrative
>   until relative-locktime support lands.
> - **`Refresh`** relies on the explicit-fee / no-change output layout with the
>   `fee` keyword. The estimate adds a fixed allowance for the covenant's Simplicity
>   witness (whose exact size is only known after signing), so it errs slightly
>   high to stay above the relay minimum — the extra just goes to the fee. Fund the
>   will with a little headroom so each refresh's fee fits.

## Try next

You've now seen multiple spending paths, a timelock, and a recursive covenant in
one file. The recipes that go deeper on those building blocks:
[Covenant UTXO types](./05-covenant-utxo-types.md) and
[Multiple spending paths](./06-multiple-spending-paths.md).
