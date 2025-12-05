# Complete Juno Cash Mining Analysis

## Block Header Structure - EXACT FORMAT

### Juno Cash Block Header Serialization

**CEquihashInput** (header without nonce and solution) - **108 bytes**:
```
Offset | Size | Field                    | Type
-------|------|--------------------------|----------
0      | 4    | nVersion                 | int32_t (little-endian)
4      | 32   | hashPrevBlock            | uint256 (internal byte order)
36     | 32   | hashMerkleRoot           | uint256 (internal byte order)
68     | 32   | hashBlockCommitments     | uint256 (internal byte order)
100    | 4    | nTime                    | uint32_t (little-endian)
104    | 4    | nBits                    | uint32_t (little-endian)
```

**Complete Mining Input** - **140 bytes total**:
```
Offset | Size | Field
-------|------|------------------
0-107  | 108  | Header (above)
108    | 32   | nNonce (uint256)
```

**RandomX Processing**:
- Input to RandomX: All 140 bytes (header + nonce)
- RandomX Output: 32 bytes
- Output stored in: `nSolution` field (separate, not part of header)
- Block is valid if: `UintToArith256(randomxHash) <= target`

### Key Insight: nNonce is 32 bytes!

Unlike Bitcoin (4-byte nonce), Juno Cash uses **uint256 (32 bytes)** for the nonce. This is critical for the miner implementation.

## getblocktemplate RPC Response

Juno Cash's getblocktemplate returns:

```json
{
  "version": 4,
  "previousblockhash": "<hex_64_chars>",
  "blockcommitmentshash": "<hex_64_chars>",
  "defaultroots": {
    "merkleroot": "<hex_64_chars>",
    "chainhistoryroot": "<hex_64_chars>",
    "authdataroot": "<hex_64_chars>",
    "blockcommitmentshash": "<hex_64_chars>"
  },
  "transactions": [
    {
      "data": "<hex_tx_data>",
      "hash": "<hex_64_chars>",
      "authdigest": "<hex_64_chars>",
      "depends": [],
      "fee": 0,
      "sigops": 0
    }
  ],
  "coinbasetxn": {
    "data": "<hex_coinbase_data>",
    "hash": "<hex_64_chars>",
    "authdigest": "<hex_64_chars>",
    "depends": [],
    "fee": 0,
    "sigops": 0,
    "required": true
  },
  "target": "<hex_64_chars>",
  "mintime": 1234567890,
  "mutable": ["time", "transactions", "prevblock"],
  "noncerange": "00000000ffffffff",
  "sigoplimit": 20000,
  "sizelimit": 2000000,
  "curtime": 1234567890,
  "bits": "1d00ffff",
  "height": 123,
  "randomxseedheight": 0,
  "randomxseedhash": "08000000000000000000000000000000000000000000000000000000000000000000000",
  "randomxnextseedhash": "<hex_64_chars>"
}
```

### Juno-Specific Fields

1. **randomxseedheight** (int): The block height whose hash is used as RandomX seed
   - Calculated via: `RandomX_SeedHeight(blockHeight)`
   - Formula: `(blockHeight / 2048) * 2048 - 96`
   - Minimum: 0 (genesis epoch)

2. **randomxseedhash** (string): The RandomX seed hash (64 hex chars)
   - For genesis epoch (height 0-2047): `"08" + 62 zeros`
   - Otherwise: Block hash at randomxseedheight
   - **Internal byte order** (not reversed like prevblockhash in Stratum)

3. **randomxnextseedhash** (string, optional): Next epoch's seed for pre-caching
   - Only present in last 96 blocks of epoch
   - Allows miners to preload next RandomX dataset

4. **blockcommitmentshash**: Zcash-specific field
   - Part of NU5 consensus rules
   - Must be included in block header at offset 68

## Address Formats

### Mainnet Addresses
- **Transparent P2PKH**: `t1...` (starts with t1, 34-36 chars)
  - Base58 prefix: `[0x1C, 0xB8]`
- **Sapling**: `zs1...` (starts with zs1, 78 chars)
  - Bech32
- **Unified**: `u1...` (starts with u1, variable length)
  - Bech32m

### Testnet Addresses
- **Transparent**: `tm...` (starts with tm, 34-36 chars)
  - Base58 prefix: `[0x1D, 0x25]`
- **Sapling**: `ztestsapling...` (78+ chars)
  - Bech32 HRP: "ztestsapling"
- **Unified**: `utest...`
  - Bech32m HRP: "utest"

### Regtest Addresses
- **Transparent**: `tm...` (same as testnet)
- **Sapling**: `ztestsapling...` (same as testnet)
- **Unified**: `uregtest...` or `jregtest...`
  - Bech32m HRP: "uregtest" or "jregtest"

## Mining Flow in Juno Cash Source

From `src/rpc/mining.cpp` (generate function):

```cpp
// 1. Get seed for current height
uint64_t blockHeight = pindexPrev->nHeight + 1;
uint64_t seedHeight = RandomX_SeedHeight(blockHeight);
uint256 seedHash = /* get from chain or use 0x08... for genesis */;

// 2. Update RandomX cache
RandomX_SetMainSeedHash(seedHash.begin(), 32);

// 3. Serialize header input
CEquihashInput I{*pblock};  // Header minus nonce and solution
CDataStream randomxInput(SER_NETWORK, PROTOCOL_VERSION);
randomxInput << I;
randomxInput << pblock->nNonce;

// 4. Hash with RandomX
uint256 randomxHash;
RandomX_Hash_Block(randomxInput.data(), randomxInput.size(), randomxHash);

// 5. Store solution
pblock->nSolution.resize(32);
memcpy(pblock->nSolution.data(), randomxHash.begin(), 32);

// 6. Check if valid
if (UintToArith256(randomxHash) <= hashTarget) {
    // Block found!
}
```

## Stratum Protocol Differences

### Our Pool Implementation

The pool server we built uses a **hybrid approach**:

**Job Format** (from pool-dev/stratum-server/src/types.ts):
```typescript
interface MiningJob {
  id: string;                    // Job ID (8 hex chars)
  height: number;                // Block height
  previousBlockHash: string;     // 64 hex chars (internal byte order)
  coinbase1: string;             // Hex coinbase part 1
  coinbase2: string;             // Hex coinbase part 2
  merkleBranches: string[];      // Merkle branches for root calculation
  version: string;               // Block version (8 hex chars)
  bits: string;                  // nBits (8 hex chars)
  timestamp: number;             // Current time
  target: string;                // 64 hex chars
  randomxSeedHeight: number;     // Juno-specific
  randomxSeedHash: string;       // Juno-specific (64 hex chars)
  cleanJobs: boolean;            // Clear old jobs
}
```

**mining.notify Parameters**:
```json
[
  "job_id",
  "version",
  "prevhash",
  "merkleroot",
  "blockcommitments",
  "time",
  "bits",
  clean_jobs,
  "randomx_seed_hash"
]
```

**mining.submit Parameters**:
```json
{
  "worker": "worker_name",
  "job_id": "job_id",
  "extra_nonce2": "hex",
  "ntime": "hex_time",
  "nonce": "64_hex_chars"  // 32 bytes!
}
```

## RandomX Seed Calculation

From `src/pow.cpp` and pool stratum server:

```cpp
uint64_t RandomX_SeedHeight(uint64_t blockHeight) {
    const uint64_t RANDOMX_EPOCH_BLOCKS = 2048;
    const uint64_t RANDOMX_EPOCH_LAG = 96;

    uint64_t epoch = blockHeight / RANDOMX_EPOCH_BLOCKS;
    uint64_t epochStartHeight = epoch * RANDOMX_EPOCH_BLOCKS;
    uint64_t seedHeight = epochStartHeight - RANDOMX_EPOCH_LAG;

    return std::max((uint64_t)0, seedHeight);
}
```

**Examples**:
- Block 0-2047: Seed height = 0 (genesis seed `0x08...`)
- Block 2048-4095: Seed height = 1952 (2048 - 96)
- Block 4096-6143: Seed height = 4000 (4096 - 96)

## Key Differences from Monero/XMRig

| Aspect | Monero | Juno Cash |
|--------|--------|-----------|
| Nonce offset | 39 | 108 |
| Nonce size | 4 bytes | **32 bytes** |
| Block blob size | ~76 bytes | 140 bytes |
| Block structure | CryptoNote | Zcash/Bitcoin |
| Stratum blob | Single hex blob | Multi-field params |
| Address prefix | 4... | t1.../zs1.../u1... |
| Extra header field | - | blockCommitments (32 bytes) |
| Solution format | Embedded | Separate nSolution field |

## Implementation Requirements for junorig

### 1. Block Construction

Miner must construct 140-byte input:
```cpp
// Bytes 0-3: version (little-endian)
// Bytes 4-35: prevBlockHash (internal order)
// Bytes 36-67: merkleRoot (internal order)
// Bytes 68-99: blockCommitments (internal order)
// Bytes 100-103: time (little-endian)
// Bytes 104-107: bits (little-endian)
// Bytes 108-139: nonce (32 bytes, uint256)
```

### 2. Nonce Management

The 32-byte nonce should be split:
```
[extraNonce1: 16 bytes][miner_nonce: 4 bytes][extraNonce2: 12 bytes]
```
- Pool assigns extraNonce1 (unique per connection)
- Miner iterates miner_nonce (0 to 0xFFFFFFFF)
- extraNonce2 from pool's mining.notify

### 3. RandomX Seed Handling

Miner must:
1. Parse `randomxseedhash` from job
2. Initialize/update RandomX VM with seed
3. Hash 140-byte block input
4. Get 32-byte output
5. Check if output <= target

### 4. Share Submission

Must submit:
```json
{
  "method": "mining.submit",
  "params": {
    "worker": "worker_name",
    "job_id": "12345678",
    "extra_nonce2": "000000000000000000000000",
    "ntime": "507f191e",
    "nonce": "32_byte_nonce_as_64_hex_chars"
  },
  "id": 4
}
```

## Testing Checklist

- [ ] Parse getblocktemplate correctly (all fields)
- [ ] Handle randomxseedheight = 0 (genesis seed)
- [ ] Handle randomxseedheight > 0 (block hash seed)
- [ ] Construct 140-byte mining input correctly
- [ ] Parse all Juno address formats (t1, tm, zs1, ztestsapling, u1, utest, uregtest, jregtest)
- [ ] Initialize RandomX with correct seed
- [ ] Generate correct 32-byte RandomX hash
- [ ] Submit shares in correct format
- [ ] Handle epoch transitions (pre-cache next seed)
- [ ] Verify block commitments field (68-99) is included

## References

- Juno Cash block header: `/Users/laid/junocash/src/primitives/block.h` (line 26-83)
- RandomX validation: `/Users/laid/junocash/src/pow.cpp` (line 132-195)
- Mining RPC: `/Users/laid/junocash/src/rpc/mining.cpp` (line 457-905)
- getblocktemplate: Lines 857-902 (RandomX fields)
- Address formats: `/Users/laid/junocash/src/key_io.h`
- Our pool Stratum: `/Users/laid/junocash/pool-dev/stratum-server/src/`
