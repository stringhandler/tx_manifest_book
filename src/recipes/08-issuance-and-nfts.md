# Asset issuance & NFTs

> 📝 **Draft.** This chapter has not been reviewed yet — content may be incomplete or change.

> **Problem.** Mint Liquid assets — including single-unit NFTs used as bearer
> tokens — as part of an action, and derive their asset IDs.

> 🚧 **This recipe is a stub.** Outline of what it will cover:
>
> - The input `issuance` spec: `kind` (`new` / `reissue`), `asset_amount_sat`,
>   `inflation_amount_sat`.
> - Why an asset ID is derived from the *outpoint* of the issuance input, and how
>   to reference the result as `input_id.asset` afterward.
> - NFTs as bearer tokens (amount = 1) authorising covenant paths.
> - Encoding protocol terms in an NFT's amount field via bit-packing
>   (the lending protocol's parameter NFTs).
> - Burning NFTs with `OP_RETURN` to prevent reuse.

See the `IssueNFTs` action in the
[lending walkthrough](../walkthrough/lending-protocol.md) in the meantime.
