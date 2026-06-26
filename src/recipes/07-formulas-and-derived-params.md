# Formulas & derived params

> **Problem.** Compute amounts, indices, and parameter values from other values
> instead of hard-coding them.

> 🚧 **This recipe is a stub.** Outline of what it will cover:
>
> - Where formulas appear: output/input `amount_sat`, validation `expr`, hook
>   `set` values, witness `expr`.
> - Operators (`+ - * /`, comparisons, `&& || !`) and references
>   (`compile_params.X`, `params.X`, `input_id.amount_sat`, `input_id.asset`,
>   `input_id.present`).
> - Functions: `pow(base, exp)`, `index_of(id)`, `concat(...)`.
> - The special `fees` value used in change formulas.
> - **Derived params** (`"derived": true`): interest = `PRINCIPAL_AMOUNT *
>   PRINCIPAL_INTEREST_RATE / 10000`, and params derived from issuance outpoints.

See [`Spec.md` §9](https://github.com/stringhandler/s-compose/blob/main/Spec.md)
for the formula grammar in the meantime.
