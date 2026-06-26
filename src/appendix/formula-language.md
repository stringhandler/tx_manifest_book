# Formula language reference

Formulas are string expressions evaluated at transaction build time. They appear
in output/input `amount_sat`, validation `expr`, hook `set` values, and witness
`expr`.

## Operators

| Operator | Description |
|----------|-------------|
| `+` `-` `*` `/` | Integer arithmetic (division truncates) |
| `==` `!=` `<` `<=` `>` `>=` | Comparison (returns boolean) |
| `&&` `\|\|` `!` | Boolean logic |
| `(` `)` | Grouping |

## References

| Syntax | Description |
|--------|-------------|
| `compile_params.NAME` | Compile parameter by name |
| `params.NAME` | Action parameter by name |
| `args.NAME` | Action argument by name |
| `input_id.amount_sat` | Satoshi amount of a resolved input |
| `input_id.asset` | Asset ID of a resolved input (hex string) |
| `input_id.present` | Boolean — whether an optional input was found |
| `output_id.amount_sat` | Satoshi amount of a constructed output (post-construction) |
| `fees` | Estimated transaction fee (used in change formulas) |

## Functions

| Function | Signature | Description |
|----------|-----------|-------------|
| `pow(base, exp)` | (u64, u64) → u64 | Integer exponentiation |
| `index_of(id)` | (input or output id) → u32 | Transaction index of a named input/output |
| `concat(a, b, …)` | (bytes…) → bytes | Byte concatenation (`OP_RETURN` `data` only) |

See [`Spec.md` §9](https://github.com/stringhandler/s-compose/blob/main/Spec.md)
for the authoritative grammar.
