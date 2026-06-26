# Introduction

Welcome to **The SimplicityCompose Cookbook** — a recipe-driven guide to writing
*compose files*: machine-readable descriptions of multi-UTXO protocols on Bitcoin
and Liquid.

This book is not a top-to-bottom reference manual. It is a **cookbook** in the
style of the [Rust Cookbook](https://rust-lang-nursery.github.io/rust-cookbook/):
a sequence of small, self-contained recipes, each solving one concrete problem
and introducing one or two new concepts. You can read it straight through, or
jump to the recipe that matches what you are trying to do.

## Who this is for

- **Protocol authors** who have a Simplicity or miniscript contract and want a
  portable description of the transactions that drive it.
- **Wallet and tooling developers** who want to read a single file and know which
  UTXOs to watch, which transactions are valid, and which witnesses to construct.
- **Anyone** trying to understand an unfamiliar on-chain protocol without
  reverse-engineering it from source.

## How the book is organised

1. **Getting Started** explains what a compose file *is*, gets the `compose-wallet`
   CLI built and a wallet ready, and dissects the top-level structure of a file.
2. **The Cookbook** is the heart of the book. Each recipe builds on the previous
   one, starting from a no-covenant warm-up (splitting a UTXO) and a single key
   locking a single output, then growing toward covenants, issuance, and
   multi-step lifecycles.
3. **The Full Walkthrough** ties every concept together on a real
   peer-to-peer lending protocol.
4. **The Appendix** is the quick-reference material: type tables, the formula
   language grammar, and the full CLI command list.

## How to read a recipe

Every recipe in the Cookbook follows the same shape:

> **Problem** — one sentence describing the goal.
>
> **Recipe** — the compose JSON you can copy and adapt.
>
> **How it works** — an annotated tour of the new fields.
>
> **Run it** — the actual `compose-wallet` commands to execute the action.
>
> **Try next** — where to go from here.

Every JSON snippet and command in this book is drawn from real files in the
repository (`p2pk_simplicity.compose.json`, `example/lending/`) and the real CLI
in `tools/compose-wallet` — nothing here is invented.

> **The authoritative reference** is [`Spec.md`](https://github.com/stringhandler/s-compose/blob/main/Spec.md)
> in the repository root. When this book and the spec disagree, the spec wins.
> This cookbook aims to teach; the spec aims to be complete.

Let's start with the big picture: [what is a compose file?](./getting-started/what-is-compose.md)
