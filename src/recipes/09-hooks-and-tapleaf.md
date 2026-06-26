# Hooks & tapleaf compute

> 📝 **Draft.** This chapter has not been reviewed yet — content may be incomplete or change.

> **Problem.** Compute and store values mid-action — especially covenant script
> hashes that depend on other covenant script hashes.

> 🚧 **This recipe is a stub.** Outline of what it will cover:
>
> - Hook blocks: `on_resolved` (per input) and `on_pre_broadcast` (per action),
>   each running `set` assignments in declaration order.
> - Assignment targets: `compile_params.X`, `params.X`, `args.X`.
> - The **tapleaf compute spec** (`lang: "tapleaf"`): compiling a `.simf` to a
>   covenant script hash, with `params` and `depends_on`.
> - **Circular dependencies**: when two covenants each reference the other's hash,
>   seed with 32 zero bytes and iterate to convergence.

See [`Spec.md` §5.5 and §11 Step 3](https://github.com/stringhandler/tx_manifest_spec/blob/main/Spec.md)
in the meantime.
