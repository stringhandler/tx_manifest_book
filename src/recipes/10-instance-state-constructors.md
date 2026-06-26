# Instance, state & constructors

> 📝 **Draft.** This chapter has not been reviewed yet — content may be incomplete or change.

> **Problem.** Deploy a contract once and then act on it repeatedly — persisting
> the compile params and tracking the live UTXO set across transactions.

> 🚧 **This recipe is a stub.** Outline of what it will cover:
>
> - The three-file model in practice: manifest / instance / state.
> - **Classes**: grouping methods under a typed contract with `fields`.
> - **Constructors** (`is_constructor: true`) and `create_instance`: writing the
>   instance file with resolved field values.
> - The **state file**: how covenant outputs are added and spent inputs removed
>   after each broadcast.
> - **`provided_inputs`**: pre-filling a counterparty's UTXO inline (the
>   website-to-wallet integration pattern).

See [`Spec.md` §12–13](https://github.com/stringhandler/tx_manifest_spec/blob/main/Spec.md)
in the meantime.
