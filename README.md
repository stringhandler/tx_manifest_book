# The tx-manifest Cookbook

A recipe-driven guide to writing **transaction manifests** — machine-readable JSON
descriptions of multi-UTXO protocols on Bitcoin and Liquid, backed by
[SimplicityHL](https://github.com/BlockstreamResearch/SimplicityHL) covenants.

This is an [mdBook](https://rust-lang.github.io/mdBook/). It is a cookbook in the
style of the [Rust Cookbook](https://rust-lang-nursery.github.io/rust-cookbook/):
a sequence of small, self-contained recipes, each solving one concrete problem and
introducing one or two new concepts, building toward a full peer-to-peer lending
protocol.

> ⚠️ **Work in progress.** The getting-started chapters, the first recipes, and the
> appendix are complete; several later recipes (05–10) and the lending walkthrough
> are still stubs being filled in.

## Building the book

You need [mdBook](https://rust-lang.github.io/mdBook/guide/installation.html):

```sh
cargo install mdbook
```

Then, from this directory:

```sh
mdbook serve --open   # live-reloading preview at http://localhost:3000
mdbook build          # render static HTML into ./book
```

## Running the recipes

The recipes are executed with the `tx-manifest-wallet` CLI (this book aliases it
to `txw`). See the [Setup](src/getting-started/setup.md) chapter for the install
options: the [Blockstream Simplicity codespace](https://github.com/Blockstream/simplicity-codespace)
(preinstalled), a [prebuilt release binary](https://github.com/stringhandler/txmanifest-wallet/releases),
the [asdf plugin](https://github.com/stringhandler/asdf-tx-manifest-wallet), or
building from source.

## Related projects

| Project | What it is |
|---------|------------|
| [`txmanifest-wallet`](https://github.com/stringhandler/txmanifest-wallet) | The reference wallet/engine (`tx-manifest-wallet`) and example manifests. |
| [`tx_manifest_spec`](https://github.com/stringhandler/tx_manifest_spec) | The authoritative specification (`Spec.md`). When the book and the spec disagree, the spec wins. |
| [`asdf-tx-manifest-wallet`](https://github.com/stringhandler/asdf-tx-manifest-wallet) | asdf plugin that installs prebuilt wallet binaries. |

## License

Licensed under either of

- Apache License, Version 2.0 ([LICENSE-APACHE](LICENSE-APACHE))
- MIT license ([LICENSE-MIT](LICENSE-MIT))

at your option.

Unless you explicitly state otherwise, any contribution intentionally submitted
for inclusion in the work by you, as defined in the Apache-2.0 license, shall be
dual licensed as above, without any additional terms or conditions.
