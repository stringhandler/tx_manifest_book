# Witnesses

> **Problem.** Provide the values a Simplicity covenant needs to authorise a
> spend — signatures and branch selectors — and understand what the tool does
> with them.

You met your first witness in
[Hello World, Part 2](./01b-hello-world-receive.md): the `SIGNATURE` that
satisfied `p2pk.simf`. This recipe steps back and covers the `witnesses` map in
full — what it is, the two kinds the tool produces, and the ones you *don't* have
to supply.

A witness only matters when you **spend** a covenant. Creating a covenant output
(`Pay`) commits to a program; nothing is checked. Spending it (`Receive`) runs the
program, and the program reads its witnesses to decide whether to allow the spend.

## Where witnesses live

Witnesses sit on an **input** — specifically a covenant input (`utxo_source` is a
`utxo_type`, not `"wallet"`). Plain wallet inputs sign themselves the ordinary
way and have no `witnesses` map.

```json
{
  "id": "p2pk_in",
  "utxo_source": { "utxo_type": "p2pk_output", "compile_params": { "PUB_KEY": "params.pubkey" } },
  "witnesses": {
    "SIGNATURE": { "type": "Signature", "sig_type": "sig_hash_all", "source": { "type": "wallet", "key": "params.pubkey" } }
  }
}
```

**Each key is a SimplicityHL witness name** — it must match a `witness::NAME` in
the `.simf` source. Our program reads `witness::SIGNATURE`, so the map has a
`SIGNATURE` entry. A program with `witness::PATH` and `witness::SIGNATURE` would
have entries for both.

## The two kinds of witness

The reference tool produces exactly two witness `type`s. (The spec sketches more;
see [Not yet wired up](#not-yet-wired-up) below.)

### `Signature` — a computed BIP340 signature

This is the one from Part 2. You don't write a signature by hand; the tool
computes it while signing.

```json
"SIGNATURE": {
  "type": "Signature",
  "sig_type": "sig_hash_all",
  "source": { "type": "wallet", "key": "params.pubkey" }
}
```

- **`sig_type: "sig_hash_all"`** selects the message to sign: Simplicity's
  `sig_all_hash`, a commitment over the whole transaction. This is **not** the
  classic Bitcoin/Elements `SIGHASH_ALL` — it's Simplicity's own hash, computed
  via the transaction environment (`CTxEnv::sighash_all()`). It's currently the
  only `sig_type` defined.
- **`source: { "type": "wallet", "key": ... }`** identifies the signing key. The
  `key` resolves to an x-only pubkey — from an action `param` (`params.pubkey`,
  as here), a compile param / class field (`compile_params.BORROWER_PUB_KEY`, the
  form the lending example uses), or a literal hex value. The tool searches your
  wallet's BIP86 derivation paths for the private key matching that pubkey and
  signs with it.

Under the hood the tool computes the hash, signs it, and **rewrites the entry as
a `simplicityhl` witness** holding the 64-byte signature as `0x…` hex — so a
`Signature` is really sugar over the next kind.

### `simplicityhl` — a literal typed value

A fixed value, parsed against the witness's type. Use it for branch selectors,
indices, and raw byte values.

```json
"PATH": {
  "type": "simplicityhl",
  "value": "Left(())",
  "simplicity_type": "Either<(), ()>",
  "description": "Take the first spending path."
}
```

- **`value`** is a SimplicityHL value expression: `Left(())` / `Right(())` to
  choose a branch of an `Either`, `0x<hex>` for a byte array, `42` for an integer.
- **`simplicity_type`** is **optional and documentary**. The tool takes the real
  type from the compiled program's ABI, not from this field — it's there to help a
  human reader. Provide it for clarity; leave it off and nothing breaks.

Branch selectors are the most common use. A covenant with two spending paths
typically reads a `witness::PATH` of type `Either<(), ()>`; supplying `Left(())`
or `Right(())` picks which path runs. That's the subject of
[Multiple spending paths](./06-multiple-spending-paths.md).

## The witnesses you *don't* supply

A program declares every witness it *could* read, but a single spend only travels
one path. You supply witnesses for the path you're taking; **any witness you omit
is filled with a zero value automatically.**

That's not a fallback for forgetfulness — it's by design. Before Simplicity prunes
the unused branches, every witness node needs *some* concrete value. Witnesses on
branches you didn't take (e.g. the `SIGNATURE` on a cancel path when you chose
`PATH = Left`) are zeroed, then pruned away and never executed. So:

> Supply only the witnesses on the path you're spending. The rest take care of
> themselves.

This is why our single-path `p2pk.simf` needs only `SIGNATURE`, and why a
two-path covenant needs `PATH` plus the witnesses for the chosen branch — not
both branches' worth.

## What the tool builds

Once witnesses are resolved, the tool satisfies the program against the spending
transaction and writes the final Simplicity tapscript witness stack — exactly four
items, in this order:

```text
[ witness_bits, pruned_program, cmr_script, control_block ]
```

You never assemble this yourself; it's the output of finalisation. The
[Simplicity dry-run](./01b-hello-world-receive.md#run-it) executes the program
against this stack before broadcast, so a missing or wrong witness is caught
locally rather than rejected by the network.

## Not yet wired up

The [spec](https://github.com/stringhandler/s-compose/blob/main/Spec.md) and some
example files reference two further witness types — **`formula`** (a computed
value such as `index_of(some_output)`) and **`taproot_leaf`** (a leaf/control-block
selector). The current reference tool **does not consume these** — only
`simplicityhl` and `Signature` are processed, and the control block is derived from
the covenant's leaf structure regardless. Treat `formula` and `taproot_leaf` as
forward-looking until the tooling catches up; if you put them on an input today,
the witness is simply zeroed like any unsupplied value.

## See also

- [`Spec.md` §8](https://github.com/stringhandler/s-compose/blob/main/Spec.md) —
  the full witness reference.
- [Multiple spending paths](./06-multiple-spending-paths.md) — using a `PATH`
  selector witness in anger.
- [Covenant UTXO types](./05-covenant-utxo-types.md) — how the covenant address
  (and its tapleaf) is derived in the first place.
