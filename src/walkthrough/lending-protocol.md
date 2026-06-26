# The lending protocol

> 📝 **Draft.** This chapter has not been reviewed yet — content may be incomplete or change.

> 🚧 **This chapter is a stub.** It will tie every cookbook concept together on a
> real peer-to-peer collateralised lending protocol on Liquid: NFT ownership,
> Simplicity covenants, partial repayments, vault accumulation, and liquidation.

The complete lending example lives in the repository at
[`examples/lending/`](https://github.com/stringhandler/txmanifest-wallet/tree/main/examples/lending)
(its [`txmanifest.json`](https://github.com/stringhandler/txmanifest-wallet/blob/main/examples/lending/txmanifest.json)
plus the covenant `.simf` programs). This chapter will adapt it into the cookbook,
cross-linking each section back to the recipe that introduced the concept:

- **Lifecycle** → [recipe 6](../recipes/06-multiple-spending-paths.md)
- **Params (compile, derived, NFT-encoded)** → recipes
  [2](../recipes/02-params-and-validations.md) &
  [7](../recipes/07-formulas-and-derived-params.md)
- **UTXO types (covenants, script-auth)** → [recipe 5](../recipes/05-covenant-utxo-types.md)
- **IssueNFTs** → [recipe 8](../recipes/08-issuance-and-nfts.md)
- **LockCollateral, SetupLending, RepayLoan, …** → recipes
  [3](../recipes/03-outputs-and-destinations.md),
  [4](../recipes/04-witnesses.md) & [9](../recipes/09-hooks-and-tapleaf.md)
