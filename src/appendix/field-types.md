# Field type reference

The type strings used in `compile_params`, class `fields`, and action `params`.

| Type string | Rust equivalent | Description |
|-------------|-----------------|-------------|
| `u8` | `u8` | 8-bit unsigned integer |
| `u16` | `u16` | 16-bit unsigned integer |
| `u32` | `u32` | 32-bit unsigned integer |
| `u64` | `u64` | 64-bit unsigned integer |
| `bytes32` | `[u8; 32]` | 32-byte raw byte array |
| `pubkey` | `[u8; 32]` | 32-byte x-only BIP340 Schnorr public key |
| `liquid.asset_id` | `[u8; 32]` | Liquid asset ID (32 bytes) |
| `address` | string | A bech32/blech32 address (used by action params) |

> Integer field values are written as decimal strings in instance files; byte
> types as hex strings. See [`Spec.md` §4.2 and §12](https://github.com/stringhandler/tx_manifest_spec/blob/main/Spec.md).
