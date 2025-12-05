# Juno Cash Modifications for junorig

This document outlines the required modifications to adapt XMRig for Juno Cash (RandomX) mining.

## Overview

Juno Cash is a Zcash fork that uses RandomX for proof-of-work instead of Equihash. The block structure and Stratum protocol differ significantly from Monero (XMRig's original target).

## Block Header Structure

### Juno Cash Block Header (140 bytes for RandomX input)

```
Offset | Size | Field
-------|------|------------------
0      | 4    | nVersion (int32_t)
4      | 32   | hashPrevBlock (uint256)
36     | 32   | hashMerkleRoot (uint256)
68     | 32   | hashBlockCommitments (uint256)
100    | 4    | nTime (uint32_t)
104    | 4    | nBits (uint32_t)
108    | 32   | nNonce (uint256) - THE MINING NONCE
```

**RandomX Input**: Bytes 0-107 (header without nonce) + bytes 108-139 (nonce) = 140 bytes total
**RandomX Output**: 32 bytes, stored as `nSolution` field
**Nonce Location**: Offset 108, size 32 bytes (uint256)

### Monero/XMRig Block Structure (for comparison)

```
Blob size: ~76 bytes (variable)
Nonce offset: 39
Nonce size: 4 bytes
```

## Key Differences from Monero

1. **Nonce size**: Juno uses 32-byte nonce (uint256) vs Monero's 4-byte nonce
2. **Nonce offset**: Juno at byte 108 vs Monero at byte 39
3. **Block header size**: Juno 140 bytes vs Monero ~76 bytes
4. **Address format**: Juno uses Zcash-style addresses (t1..., zs1..., u1..., jregtest...) vs Monero addresses
5. **Stratum params**: Different job structure

## Stratum Protocol Differences

### Job Notification (mining.notify)

**Monero/XMRig format**:
```json
{
  "params": {
    "blob": "<hex_mining_blob>",
    "job_id": "<id>",
    "target": "<hex_target>",
    "seed_hash": "<hex_seed>"
  }
}
```

**Juno Cash format** (Bitcoin/Zcash-style):
```json
{
  "params": [
    "<job_id>",
    "<version>",
    "<previous_block_hash>",
    "<merkle_root>",
    "<block_commitments>",
    "<time>",
    "<bits>",
    "<clean_jobs>",
    "<randomx_seed_hash>"
  ]
}
```

### Share Submission (mining.submit)

**XMRig/Monero**:
```json
{
  "params": {
    "id": "<worker_id>",
    "job_id": "<id>",
    "nonce": "<4_byte_hex_nonce>",
    "result": "<32_byte_hash>"
  }
}
```

**Juno Cash**:
```json
{
  "params": {
    "worker": "<worker_name>",
    "job_id": "<id>",
    "extra_nonce2": "<hex>",
    "ntime": "<hex_time>",
    "nonce": "<32_byte_hex_nonce>"
  }
}
```

## Files Requiring Modification

### 1. Job Structure (`src/base/net/stratum/Job.h` and `Job.cpp`)

**Changes needed**:
- Modify `nonceOffset()` to return 108 for Juno Cash
- Update `nonceSize()` to return 32 for Juno Cash
- Change `kMaxBlobSize` or add Juno-specific constant (140 bytes minimum)
- Add Juno Cash algorithm detection
- Modify `setBlob()` to handle Zcash-style job construction

**New method needed**:
```cpp
bool setJobFromZcashParams(
    const char* version,
    const char* prevHash,
    const char* merkleRoot,
    const char* blockCommitments,
    uint32_t time,
    const char* bits
);
```

### 2. Stratum Client (`src/base/net/stratum/Client.cpp`)

**Changes needed**:
- Modify job notification handler to parse Zcash-style params
- Update `submit()` to send Zcash-style submission with:
  - 32-byte nonce instead of 4-byte
  - extra_nonce2 parameter
  - ntime parameter
- Add Juno Cash protocol detection

### 3. Algorithm Detection (`src/base/crypto/Algorithm.h` and `Algorithm.cpp`)

**Add new algorithm**:
```cpp
enum Id : int {
    // ... existing ...
    RX_JUNO,  // RandomX for Juno Cash
};
```

### 4. Nonce Handling

**Current XMRig behavior**:
- Uses 4-byte nonce at offset 39
- Iterates through nonce space rapidly

**Juno Cash requirement**:
- Uses 32-byte nonce at offset 108
- Only need to use first 4 bytes for iteration (rest can be extraNonce)
- Layout: `[extraNonce1 (16 bytes)][miner_nonce (4 bytes)][extraNonce2 (12 bytes)]`

### 5. Share Validator Integration

The pool already has RandomX validation via the standalone hasher. The miner just needs to:
1. Construct proper 140-byte header
2. Submit with correct format
3. Let pool validate with RandomX

## Implementation Strategy

### Phase 1: Minimal Working Implementation
1. Add `RX_JUNO` algorithm constant
2. Modify `nonceOffset()` to return 108 for RX_JUNO
3. Modify `nonceSize()` to return 32 for RX_JUNO (but only iterate first 4 bytes)
4. Update job parsing for Zcash-style params
5. Update submission format

### Phase 2: Full Integration
1. Add proper address validation for Juno Cash
2. Implement full nonce space management
3. Add configuration file support for Juno Cash pools
4. Update documentation

### Phase 3: Optimization
1. Optimize RandomX cache handling
2. Add Juno-specific benchmarking
3. Performance tuning

## Configuration Example

```json
{
  "pools": [
    {
      "url": "stratum+tcp://pool.junojunkies.org:3333",
      "user": "jregtest1wkcaqf7xksj7v5laqf8ra77rfwlurall6cx30w4xqad0nm3tmqdajeafs9su3uqmru8k7jh8rpn77wrr645w9yt47dk9cf92kqq2wreq",
      "pass": "x",
      "algo": "rx/juno"
    }
  ],
  "randomx": {
    "mode": "auto",
    "1gb-pages": false
  }
}
```

## Testing Plan

1. **Unit tests**: Verify nonce offset, size, blob construction
2. **Pool connection**: Test subscribe/authorize with pool server
3. **Job reception**: Parse and validate job notifications
4. **Share submission**: Submit shares in correct format
5. **Hash validation**: Verify RandomX hashes match pool expectations
6. **End-to-end**: Mine on regtest and find blocks

## RandomX Compatibility

Good news: XMRig already has full RandomX support! We just need to:
- Point it at the correct blob structure
- Use correct nonce position/size
- Handle Zcash-style Stratum protocol

The RandomX VM, cache management, and hashing are already implemented and battle-tested.

## Reference

- Juno Cash block header: `src/primitives/block.h`
- RandomX validation: `src/pow.cpp` (CheckRandomXSolution)
- Pool Stratum implementation: `pool-dev/stratum-server/src/`
- XMRig RandomX: `src/crypto/randomx/`
