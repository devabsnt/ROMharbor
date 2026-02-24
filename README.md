# ROMharbor

Decentralized ROM storage on Ethereum. Upload, browse, and download ROM files entirely on-chain. The frontend itself is stored on-chain via SSTORE2 and loaded by a minimal bootstrap HTML file.

## Live

- **Mainnet contract:** [`0x39421f0da6F737cc6b6ee0eC5fcC4C4aEf7cAD0b`](https://etherscan.io/address/0x39421f0da6F737cc6b6ee0eC5fcC4C4aEf7cAD0b)

## How it works

```
User opens bootstrap.html locally (or uses the static page)
        |
        v
Bootstrap fetches frontend chunks from contract (SSTORE2)
        |
        v
Full app.html renders in browser
        |
        v
Browse/upload/download ROMs via event logs
```

### Storage architecture

- **Frontend (app.html):** Stored on-chain via SSTORE2 — data is written as contract bytecode. Split into 24KB chunks.
- **Bootstrap (bootstrap.html):** A ~2.5KB loader that fetches and renders the on-chain frontend. Hosted anywhere (GitHub Pages, local file, IPFS) or read directly from the contract via `getFrontend()`.
- **ROM data:** Stored as event logs (`ROMData` events), which are ~25x cheaper than contract storage. Permanent and readable via `eth_getLogs`.
- **ROM metadata:** Stored in contract storage (name, uploader, size, finalized status).

### Upload flow

1. ROM file is compressed in-browser using DEFLATE-raw (`CompressionStream` API)
2. For small ROMs (<120KB compressed): single `uploadROM()` transaction
3. For large ROMs: `createROM()` + multiple `addROMData()` + `finalizeROM()`, with nonce pipelining (batches of 16 pre-signed transactions)

### Download flow

1. Fetch ROM metadata via `getROM(romId)`
2. Query `ROMData` event logs from `blockStart` to get compressed chunks
3. Reassemble and decompress in-browser

## Fetching the frontend directly

You don't need the hosted bootstrap. The entire frontend can be reconstructed from any Ethereum RPC.

### Function signatures

| Function | Selector | Returns |
|------------------------------|--------------|---------------------------|
| `getFrontendChunkCount()`    | `0xdfe654b1` | `uint256`                 |
| `readFrontendChunk(uint256)` | `0x6e680e0c` | `bytes`                   |
| `getFrontend()`              | `0x17fc45e2` | `string` (bootstrap HTML) |

### Fetch with cast

```bash
CONTRACT=0x39421f0da6F737cc6b6ee0eC5fcC4C4aEf7cAD0b
RPC=https://ethereum-rpc.publicnode.com

# Get number of frontend chunks
cast call $CONTRACT "getFrontendChunkCount()(uint256)" --rpc-url $RPC

# Read chunk 0
cast call $CONTRACT "readFrontendChunk(uint256)(bytes)" 0 --rpc-url $RPC

# Read chunk 1
cast call $CONTRACT "readFrontendChunk(uint256)(bytes)" 1 --rpc-url $RPC

# Get bootstrap HTML directly from chain
cast call $CONTRACT "getFrontend()(string)" --rpc-url $RPC
```

### Fetch with JavaScript

```javascript
const CONTRACT = "0x39421f0da6F737cc6b6ee0eC5fcC4C4aEf7cAD0b";
const RPC = "https://ethereum-rpc.publicnode.com";

async function call(data) {
  const r = await fetch(RPC, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      jsonrpc: "2.0", id: 1, method: "eth_call",
      params: [{ to: CONTRACT, data }, "latest"]
    })
  });
  return (await r.json()).result;
}

// Get chunk count
const countHex = await call("0xdfe654b1");
const count = Number(BigInt(countHex));

// Read and decode each chunk
const chunks = [];
for (let i = 0; i < count; i++) {
  const idx = i.toString(16).padStart(64, "0");
  const result = await call("0x6e680e0c" + idx);
  const data = result.slice(2);
  const offset = parseInt(data.slice(0, 64), 16);
  const len = parseInt(data.slice(offset * 2, offset * 2 + 64), 16);
  const hex = data.slice(offset * 2 + 64, offset * 2 + 64 + len * 2);
  const bytes = new Uint8Array(len);
  for (let j = 0; j < len; j++)
    bytes[j] = parseInt(hex.substr(j * 2, 2), 16);
  chunks.push(bytes);
}

// Combine chunks into the full frontend HTML
const total = chunks.reduce((s, c) => s + c.length, 0);
const combined = new Uint8Array(total);
let pos = 0;
for (const c of chunks) { combined.set(c, pos); pos += c.length; }
const html = new TextDecoder().decode(combined);
```

## Contract ABI

```solidity
// Upload (single tx, small ROMs)
function uploadROM(string name, bytes data) external payable returns (uint256 romId)

// Upload (multi-tx, large ROMs)
function createROM(string name) external payable returns (uint256 romId)
function addROMData(uint256 romId, bytes data) external
function finalizeROM(uint256 romId) external

// Read
function getROM(uint256 romId) external view returns (
    string name, address uploader, uint256 totalSize, uint256 blockStart, bool finalized
)
function getROMCount() external view returns (uint256)
function getAllROMs() external view returns (
    uint256[] ids, string[] names, address[] uploaders,
    uint256[] sizes, uint256[] blockStarts, bool[] finalizedFlags
)

// Frontend
function getFrontendChunkCount() external view returns (uint256)
function readFrontendChunk(uint256 chunkIndex) external view returns (bytes)
function getFrontend() external view returns (string)

// Admin
function uploadPrice() external view returns (uint256)  // 0.005 ETH
function owner() external view returns (address)

// Events
event ROMCreated(uint256 indexed romId, string name, address uploader)
event ROMData(uint256 indexed romId, bytes data)
event ROMFinalized(uint256 indexed romId, uint256 totalSize)
```

## Cost breakdown

At ~0.05 Gwei:

| Operation | Gas | ETH |
|--------------------------------|------|---------|
| ROM upload (per MB compressed) | ~23M | ~0.0012 |
| Upload fee (per ROM)           |  --  | 0.005   |

ROM data is stored as event logs (~375 gas/byte), which are ~25x cheaper than SSTORE2 for write-heavy data while remaining permanently accessible via `eth_getLogs`.
