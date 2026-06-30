# Multiple spending paths

> 📝 **Draft.** This chapter has not been reviewed yet — content may be incomplete or change.

> **Problem.** Build a covenant that can be spent in more than one way — e.g. a
> cooperative path and a cancel/timeout path — and select between them at spend
> time.

> 🚧 **This recipe is a stub.** Outline of what it will cover:
>
> - A Simplicity program with `Either<(), ()>` paths (`PATH::LEFT` / `PATH::RIGHT`).
> - Selecting a path with a `simplicityhl` witness: `Left(())` vs `Right(())`.
> - Worked example: the lending `pre_lock` covenant — `SetupLending` takes the
>   left path; `CancelOffer` takes the right path with a borrower signature.
> - Using `required_index` on inputs so the covenant's introspection lines up.
> - Cooperative vs unilateral paths, and how they show up in `lifecycle`.

See the `pre_lock` discussion in
[Accepting or cancelling the offer](../walkthrough/lending-accept.md) (and the
`lending` covenant in [Settling the loan](../walkthrough/lending-settlement.md))
in the meantime.
