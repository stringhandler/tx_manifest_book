# Glossary

**Action / Method** — A single transaction recipe in a compose file: its inputs,
outputs, witnesses, and validations. *Action* (top-level) and *method* (inside a
class) are structurally identical.

**Attestation** — A BIP340 signature over the finalized compose document by a
developer, auditor, or counterparty. Tampering invalidates it.

**Class** — A typed contract definition with named `fields` and `methods`. Each
deployed instance of a class has its own instance file.

**CMR (Commitment Merkle Root)** — The 32-byte hash of a compiled Simplicity
program. Doubles as the program's on-chain identity.

**`canonical_cmr`** — The CMR of a program with all parameters zeroed. A stable
identifier for the program's *structure*, independent of instance parameters.

**Compile param** — A value baked into a covenant script at deploy time. Changing
one changes the script's address. Stored in the instance file.

**Compose file** — The static JSON protocol definition (`<name>.compose.json`).

**Covenant** — A script that constrains how its output may be spent — e.g. by
introspecting the spending transaction's inputs and outputs.

**Derived param** — A compile param computed by the tool rather than supplied:
from a formula, or from the outpoint of an issuance input.

**Instance file** — Per-deployment compile params and class field values
(`<name>.instance.json`).

**NUMS point** — "Nothing Up My Sleeve" — a public key with no known private key,
used as a Taproot internal key to make the key-path provably unspendable.

**`provided_inputs`** — UTXOs pre-filled inline in the instance file, letting a
wallet spend a counterparty's output it never indexed.

**PSET** — Partially Signed Elements Transaction (the Elements equivalent of a
PSBT).

**State file** — The live on-chain UTXO set for one instance
(`<name>.state.json`), updated after every broadcast.

**Tapleaf compute spec** — A field value (`lang: "tapleaf"`) that compiles a
`.simf` file with params to produce a covenant script hash.

**UTXO type** — A named on-chain state with a known script, so a wallet can
recognise the protocol's outputs.

**Witness** — A value supplied to satisfy a Simplicity program when spending:
a signature, a path selector, a leaf selector, or a computed value.
