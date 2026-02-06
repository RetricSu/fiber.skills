# Fiber On-Chain Scripts Reference

This document provides detailed technical reference for Fiber's on-chain smart contracts.

## Overview

Fiber uses two main on-chain scripts deployed on CKB:
1. **Funding Lock** - Secures channel funding with 2-of-2 MuSig2 multisig
2. **Commitment Lock** - Manages channel state, TLCs, and settlements

Both scripts are written in Rust using `ckb-std` and compiled to RISC-V.

## Testnet Deployment

| Contract | Type ID | Data Hash |
|----------|---------|-----------|
| auth | N/A (dependency) | `0x07f2364fe08ef796277d85ead9f44140b5c60feb7a6332f66c6794fdefc3645b` |
| funding-lock | `0x6c67887fe201ee0c7853f1682c0b77c0e6214044c156c7558269390a8afa6d7c` | `0x148d9176f78bc13d9d2ec65e3630df9fd6a2019364463dbc5ca69bcf660540c0` |
| commitment-lock | `0x740dee83f87c6f309824d8fd3fbdd3c8380ee6fc9acc90b1a748438afcdf81d8` | `0xff746c652d4c4852fd54d4494cfbde6f733c117618e993f43022237552e4f920` |

## Funding Lock Script

### Purpose
The funding lock secures the initial channel funding transaction. It requires both parties to sign to spend the funds, implemented via MuSig2 Schnorr signature aggregation.

### Script Args (20 bytes)
```
pubkey_hash: [u8; 20]  // Blake160 hash of x-only aggregated public key
```

### Witness Format (112 bytes)
```
EMPTY_WITNESS_ARGS: [u8; 16]  // Fixed: 0x10000000100000001000000010000000 (xUDT compatibility)
pubkey: [u8; 32]               // X-only aggregated public key
signature: [u8; 64]            // Aggregated Schnorr signature
```

### Validation Logic
1. Verify only one input uses this lock (prevents replay)
2. Verify witness is exactly 112 bytes
3. Verify first 16 bytes are EMPTY_WITNESS_ARGS placeholder
4. Extract pubkey and signature from witness
5. Compute message = blake2b256(raw_tx without cell_deps)
6. Verify Schnorr signature using `auth` library (algorithm ID = 7)

### Error Codes
| Code | Name | Description |
|------|------|-------------|
| 5 | MultipleInputs | More than one input with this lock |
| 6 | WitnessLenError | Witness not exactly 112 bytes |
| 7 | EmptyWitnessArgsError | First 16 bytes not EMPTY_WITNESS_ARGS |
| 8 | AuthError | Signature verification failed |

## Commitment Lock Script

### Purpose
The commitment lock enforces the latest channel state and handles:
- Revocation (punishing old state broadcasts)
- TLC settlements (preimage and timeout paths)
- Party settlements (local/remote fund withdrawal)
- Support for both CKB native assets and SUDT tokens

### Script Args (57 bytes)
```rust
struct CommitmentLockArgs {
    revocation_pubkey_hash: [u8; 20],  // bytes 0-20: Blake160 of x-only aggregated pubkey
    delay_epoch: u64,                   // bytes 20-28: Since format relative epoch lock
    version: u64,                       // bytes 28-36: Commitment transaction version (big-endian)
    settlement_script_hash: [u8; 20],   // bytes 36-56: Blake160 of settlement script
    is_first_settlement: u8,            // byte 56: 0x00 = first, 0x01 = subsequent
}
```

### Witness Format

#### Revocation Unlock (unlock_count = 0x00)
```
EMPTY_WITNESS_ARGS: [u8; 16]
unlock_count: u8 = 0x00
new_version: [u8; 8]           // Must be >= current version
pubkey: [u8; 32]               // X-only aggregated pubkey
signature: [u8; 64]            // Schnorr signature
```

#### Settlement Unlock (unlock_count > 0)
```
EMPTY_WITNESS_ARGS: [u8; 16]
unlock_count: u8                              // Number of settlement unlocks
pending_tlc_count: u8                         // Number of pending TLCs
tlc_scripts: [TlcScript; pending_tlc_count]   // 85 bytes each
settlement_remote_pubkey_hash: [u8; 20]       // Remote party's settlement address
settlement_remote_amount: u128                // Remote party's balance (little-endian)
settlement_local_pubkey_hash: [u8; 20]        // Local party's settlement address
settlement_local_amount: u128                 // Local party's balance (little-endian)
settlements: [Settlement; unlock_count]       // Variable length
```

### TLC Script Structure (85 bytes)
```rust
struct TlcScript {
    flags: u8,                          // bit 0: type (0=offered, 1=received)
                                        // bit 1: hash_type (0=blake2b, 1=sha256)
    payment_amount: u128,               // 16 bytes, little-endian
    payment_hash: [u8; 20],             // First 20 bytes of hash
    remote_tlc_pubkey_hash: [u8; 20],   // Blake160 of remote TLC key
    local_tlc_pubkey_hash: [u8; 20],    // Blake160 of local TLC key
    tlc_expiry: u64,                    // Since format epoch (absolute timestamp)
}
```

### Settlement Structure
```rust
struct Settlement {
    unlock_type: u8,      // 0x00-0xFD = TLC index, 0xFE = remote settlement, 0xFF = local settlement
    with_preimage: u8,    // 0x00 = no preimage, 0x01 = has preimage
    signature: [u8; 65],  // secp256k1 signature (recoverable)
    preimage: [u8; 32],   // Only present if with_preimage = 0x01
}
```

### Unlock Flows

#### 1. Revocation Flow
Used when counterparty broadcasts an old commitment transaction.

```
1. Check unlock_count == 0x00
2. Verify new_version >= current_version (from args)
3. Compute message = blake2b256(output[0] || output_data_len || output_data || pubkey_hash || new_version)
4. Verify Schnorr signature against revocation_pubkey_hash
5. Verify output has commitment_lock with updated args
```

#### 2. TLC Preimage Settlement
Used to claim funds by revealing the payment preimage.

**For Offered TLC** (tlc_type bit[0] = 0):
```
1. Verify input since == 0 (no timelock required)
2. For first settlement: verify since >= delay_epoch * 1/3
3. Verify hash(preimage)[0..20] == payment_hash
4. Verify signature from remote_tlc_pubkey_hash
5. Deduct payment_amount from channel balance
```

**For Received TLC** (tlc_type bit[0] = 1):
```
1. Verify input since == 0 (no timelock required)
2. For first settlement: verify since >= delay_epoch * 1/3
3. Verify hash(preimage)[0..20] == payment_hash
4. Verify signature from local_tlc_pubkey_hash
5. Deduct payment_amount from channel balance
```

#### 3. TLC Expiry Settlement
Used to reclaim funds after TLC timeout.

**For Offered TLC** (tlc_type bit[0] = 0):
```
1. Verify input since >= tlc_expiry (absolute timestamp)
2. For first settlement: verify since >= delay_epoch * 2/3
3. Verify signature from local_tlc_pubkey_hash
4. Deduct payment_amount from channel balance
```

**For Received TLC** (tlc_type bit[0] = 1):
```
1. Verify input since >= tlc_expiry (absolute timestamp)
2. For first settlement: verify since >= delay_epoch * 2/3
3. Verify signature from remote_tlc_pubkey_hash
4. Deduct payment_amount from channel balance
```

#### 4. Party Settlement
Used for final withdrawal of local or remote party funds.

**Remote Settlement** (unlock_type = 0xFE):
```
1. Verify since >= delay_epoch (full delay for first settlement)
2. Verify signature from settlement_remote_pubkey_hash
3. Deduct settlement_remote_amount from channel balance
```

**Local Settlement** (unlock_type = 0xFF):
```
1. Verify since >= delay_epoch (full delay for first settlement)
2. Verify signature from settlement_local_pubkey_hash
3. Deduct settlement_local_amount from channel balance
```

### Time Lock Windows

The contract enforces graduated time locks for different unlock scenarios:

| Unlock Type | First Settlement Delay | Subsequent Settlement |
|-------------|------------------------|----------------------|
| TLC Preimage | 1/3 × delay_epoch | No delay |
| TLC Expiry | 2/3 × delay_epoch | No delay |
| Party Settlement | 1 × delay_epoch | No delay |

This tiered approach allows:
- Quick preimage-based settlements (earliest window)
- Medium-delay timeout paths
- Full-delay party settlements (latest window)

### Output Validation
When creating a new commitment cell (not final settlement):

1. **Lock script validation**:
   - Output lock must be commitment lock
   - Same code_hash and hash_type as input
   - Updated settlement_script_hash (blake160 of new settlement script)
   - is_first_settlement = 0x01 (mark as subsequent)

2. **For UDT channels**:
   - Output capacity must equal input capacity
   - Output type script must match input type script
   - Output UDT amount (first 16 bytes of data) must equal new_amount

3. **For CKB channels**:
   - Output capacity must equal new_amount

### Settlement Script Hash Verification
```rust
// The settlement script in witness must hash to the value in args
if blake2b_256(&witness[settlement_script_start..settlement_script_end])[0..20]
   != args[36..56] {
    return Err(Error::WitnessHashError);
}
```

### Preimage Hash Verification
Supports two hash algorithms based on TLC flags:
```rust
match tlc.payment_hash_type() {
    PaymentHashType::Blake2b => {
        tlc.payment_hash() == &blake2b_256(preimage)[0..20]
    }
    PaymentHashType::Sha256 => {
        tlc.payment_hash() == &sha256(preimage)[0..20]
    }
}
```

### Error Codes
| Code | Name | Description |
|------|------|-------------|
| 5 | MultipleInputs | More than one input in script group |
| 6 | InvalidSince | Since value doesn't meet time requirements |
| 7 | InvalidUnlockType | Unknown unlock type (not TLC index, 0xFE, or 0xFF) |
| 8 | InvalidWithPreimageFlag | with_preimage not 0x00 or 0x01 |
| 9 | InvalidSettlementCount | Settlement count mismatch |
| 10 | InvalidUnlockCount | No settlements provided (unlock_count = 0 in settlement mode) |
| 11 | InvalidExpiry | TLC expiry not reached for timeout settlement |
| 12 | ArgsLenError | Args not exactly 57 bytes |
| 13 | WitnessLenError | Witness too short for required data |
| 14 | EmptyWitnessArgsError | First 16 bytes not EMPTY_WITNESS_ARGS |
| 15 | WitnessHashError | Settlement script hash doesn't match args |
| 16 | InvalidFundingTx | Invalid funding transaction reference |
| 17 | InvalidRevocationVersion | New version not >= current version |
| 18 | OutputCapacityError | Output capacity doesn't match expected |
| 19 | OutputLockError | Output lock script invalid |
| 20 | OutputTypeError | Output type script doesn't match input |
| 21 | OutputUdtAmountError | UDT amount doesn't match expected |
| 22 | PreimageError | Preimage hash doesn't match payment_hash |
| 23 | AuthError | Signature verification failed |

## Cryptographic Primitives

### Hash Functions
- **Blake2b-256**: Used for transaction hashing, settlement script hashing, and optional TLC payment hashes
- **SHA-256**: Optional for TLC payment hashes (for Bitcoin Lightning interoperability)

### Signature Schemes
- **Schnorr (algorithm ID 7)**: Used for funding lock and revocation unlock
  - MuSig2 aggregated signatures
  - 64-byte signature format
- **Secp256k1 ECDSA (algorithm ID 0)**: Used for TLC and party settlements
  - 65-byte recoverable signature format

### Since Format
Fiber uses CKB's `Since` type for timelocks:
- **Relative epoch-based delays**: For settlement windows (delay_epoch)
- **Absolute timestamp**: For TLC expiry

## Security Considerations

1. **Single Input Constraint**: Both scripts enforce single input to prevent replay attacks
2. **Version Monotonicity**: Revocation requires new_version >= current_version
3. **Delay Windows**: Graduated delays (1/3, 2/3, full) prevent race conditions between different unlock paths
4. **Hash Truncation**: Using 20-byte hashes (Blake160) balances security and space efficiency
5. **xUDT Compatibility**: EMPTY_WITNESS_ARGS placeholder ensures compatibility with xUDT standard
6. **Settlement Script Commitment**: The settlement_script_hash in args commits to the current channel state

## Channel Lifecycle with Scripts

### 1. Channel Opening
```
┌─────────────────────────────────────────┐
│           Funding Transaction           │
├─────────────────────────────────────────┤
│  Output: Funding Lock Cell              │
│  - Lock: funding-lock                   │
│  - Args: blake160(aggregated_pubkey)    │
│  - Capacity: total_funding_amount       │
└─────────────────────────────────────────┘
```

### 2. State Updates (Off-chain)
- TLCs added/removed
- Party balances updated
- New settlement scripts created
- Commitment transactions signed but not broadcast

### 3. Force Close (Commitment Transaction)
```
┌─────────────────────────────────────────┐
│        Commitment Transaction           │
├─────────────────────────────────────────┤
│  Input: Funding Lock Cell               │
│  - Witness: aggregated Schnorr sig      │
├─────────────────────────────────────────┤
│  Output: Commitment Lock Cell           │
│  - Lock: commitment-lock                │
│  - Args: 57 bytes (see above)           │
│  - Capacity: channel_balance            │
└─────────────────────────────────────────┘
```

### 4. Settlement Transactions
```
┌─────────────────────────────────────────┐
│         Settlement Transaction          │
├─────────────────────────────────────────┤
│  Input: Commitment Lock Cell            │
│  - Witness: settlement data + sigs      │
├─────────────────────────────────────────┤
│  Output: New Commitment Lock Cell       │
│  (if partial settlement)                │
│  OR                                     │
│  Output: User's Lock Script             │
│  (if final settlement)                  │
└─────────────────────────────────────────┘
```

### 5. Revocation (Penalty Transaction)
```
┌─────────────────────────────────────────┐
│          Penalty Transaction            │
├─────────────────────────────────────────┤
│  Input: Old Commitment Lock Cell        │
│  - Witness: revocation unlock           │
│  - new_version >= old_version           │
├─────────────────────────────────────────┤
│  Output: Punisher's Lock Script         │
│  - All funds go to honest party         │
└─────────────────────────────────────────┘
```

## SUDT/xUDT Token Support

The commitment-lock handles both native CKB and User-Defined Tokens:

**Native CKB**:
- Uses cell capacity for balance tracking
- Data field unused or empty

**SUDT/xUDT Tokens**:
- Reads amount from cell data (first 16 bytes as u128 little-endian)
- Verifies type_script matches between input and output
- Validates output UDT amount equals calculated new_amount

```rust
// UDT amount validation
if let Some(udt_script) = type_script {
    let input_amount = u128::from_le_bytes(input_data[0..16]);
    let output_amount = u128::from_le_bytes(output_data[0..16]);
    if output_amount != new_amount {
        return Err(Error::OutputUdtAmountError);
    }
}
```

## References

- [Fiber Scripts Repository](https://github.com/nervosnetwork/fiber-scripts)
- [CKB Script Programming](https://docs.nervos.org/docs/basics/guides/crypto%20wallets/ckb-script-programming)
- [ckb-std Library](https://github.com/nervosnetwork/ckb-std)
- [CKB Auth Library](https://github.com/nervosnetwork/ckb-auth)
