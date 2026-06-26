# Covenant UTXO types

> **Problem.** Define an on-chain state whose address is a Taproot output built
> from one or more Simplicity programs.

> 🚧 **This recipe is a stub.** Outline of what it will cover:
>
> - The `script` block as the tool reads it: `type: "simplicity"`, a `source`
>   path to the `.simf` file, and a `compile_params` map wiring compose params
>   onto the program's `param::*` names (e.g. `{ "PUB_KEY": "PUBKEY" }`).
> - How the tool turns that into an address: compile the `.simf` → CMR → a Taproot
>   output with a `NUMS` internal key, so the key-path is unspendable and every
>   spend goes through the script.
> - `canonical_cmr`: the CMR with params zeroed — a stable identifier a wallet uses
>   to recognize the program independent of instance parameters.
> - Covenant address determinism: same `.simf` + same params → same address,
>   always. The `P2TR(NUMS, tapbranch(...))` construction.
> - `extra_leaves` for appending additional taproot leaves.

See [`Spec.md` §14 "Covenant Address Determinism"](https://github.com/stringhandler/s-compose/blob/main/Spec.md)
in the meantime.
