---
name: fiber-network
description: Build on Fiber Network for CKB. Covers payment channels, PTLCs (Point Time-Locked Contracts), multi-hop routing, multi-path payments, trampoline routing, cross-chain hub (BTC Lightning interop), RPC APIs, on-chain scripts (funding-lock, commitment-lock), and UDT/stablecoin support. Use when building Layer 2 payment applications, integrating with Fiber nodes, or understanding CKB payment channels.
license: MIT
metadata:
  author: nervosnetwork
  version: "2.0"
  repository: https://github.com/nervosnetwork/fiber
---

# Fiber Network - Layer 2 Payment Network on CKB

## Overview

Fiber Network is a next-generation Layer 2 payment network built on Nervos CKB. It enables fast, low-cost, decentralized multi-token payments and peer-to-peer transactions for CKB and RGB++ assets (including stablecoins like RUSD).

**Key Performance**:
- Ultra-low latency: ~20ms for transfers
- Micropayment support: down to 0.0001 cent
- Microsecond-level fees: down to 0.00000001 cent

## Key Concepts

### What is Fiber?
- **Layer 2 scaling solution** for CKB blockchain
- **Payment channels** for instant off-chain transactions
- **Multi-hop routing** for payments across the network
- **Multi-asset support** - native CKB and UDT tokens (RGB++ assets)
- **Bitcoin Lightning Network interoperability** via Cross-Chain Hub (CCH)
- **PTLC-based** (Point Time-Locked Contracts) for improved privacy over HTLCs

### Core Components
1. **Fiber Node (fnn)** - The node software that manages channels and payments
2. **Funding Lock Script** - On-chain script for channel funding (2-of-2 MuSig2 multisig)
3. **Commitment Lock Script** - On-chain script for commitment transactions with TLC support
4. **Payment Channels** - Off-chain state channels between two parties
5. **PTLCs** - Point Time-Locked Contracts for secure multi-hop payments
6. **Cross-Chain Hub (CCH)** - Bridge for BTC Lightning ↔ CKB Fiber interoperability
7. **Watchtower** - Service for monitoring channels and preventing fraud

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Fiber Node (fnn)                              │
├─────────────────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────────────┐ │
│  │   Channel    │  │   Payment    │  │         Network            │ │
│  │    Actor     │  │   Session    │  │    (P2P via Tentacle)      │ │
│  └──────────────┘  └──────────────┘  └────────────────────────────┘ │
├─────────────────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────────────┐ │
│  │   Invoice    │  │    Graph     │  │        Watchtower          │ │
│  │   Manager    │  │  (Routing)   │  │         Service            │ │
│  └──────────────┘  └──────────────┘  └────────────────────────────┘ │
├─────────────────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────────────┐ │
│  │    Gossip    │  │ Cross-Chain  │  │      Onion Routing         │ │
│  │   Protocol   │  │  Hub (CCH)   │  │     (fiber-sphinx)         │ │
│  └──────────────┘  └──────────────┘  └────────────────────────────┘ │
├─────────────────────────────────────────────────────────────────────┤
│                        JSON-RPC API (port 8227)                      │
└─────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      CKB Blockchain (L1)                             │
│  ┌─────────────────────────┐  ┌───────────────────────────────────┐ │
│  │      Funding Lock       │  │       Commitment Lock             │ │
│  │   (2-of-2 MuSig2)       │  │   (TLC + revocation + settlement) │ │
│  └─────────────────────────┘  └───────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

### Actor Model Architecture

Fiber uses the Ractor actor framework for concurrent message-passing:

| Actor | Purpose |
|-------|---------|
| **RootActor** | Supervision root for all child actors |
| **NetworkActor** | Core P2P protocol handler, message routing |
| **ChannelActor** | Per-channel state machine (one per channel) |
| **PaymentActor** | Payment session tracking and lifecycle |
| **GossipActor** | Network graph synchronization |
| **CkbChainActor** | Blockchain interaction and tx monitoring |
| **WatchtowerActor** | Channel breach monitoring |
| **CchActor** | Cross-chain hub operations |

## Channel Lifecycle

### 1. Opening a Channel
```
Alice                                    Bob
  │                                       │
  │──── OpenChannel (funding_amount) ────▶│
  │◀─── AcceptChannel (funding_amount) ───│
  │                                       │
  │──── CommitmentSigned ────────────────▶│
  │◀─── CommitmentSigned ────────────────│
  │                                       │
  │──── TxSignatures ───────────────────▶│
  │◀─── TxSignatures ───────────────────│
  │                                       │
  │     [Funding TX confirmed on-chain]   │
  │                                       │
  │──── ChannelReady ───────────────────▶│
  │◀─── ChannelReady ───────────────────│
  │                                       │
  │     [Channel is now operational]      │
```

### 2. Making Payments (TLC Flow)
```
Sender                  Router                  Receiver
  │                       │                        │
  │                       │◀── new_invoice ────────│
  │◀── invoice ───────────│                        │
  │                       │                        │
  │── send_payment ──────▶│                        │
  │                       │── AddTlc (PTLC) ──────▶│
  │                       │◀── CommitmentSigned ──│
  │◀── CommitmentSigned ──│                        │
  │                       │                        │
  │                       │◀── RemoveTlc(preimage)│
  │◀── RemoveTlc ────────│                        │
  │                       │                        │
  │   [Payment complete]  │                        │
```

### 3. Closing a Channel
- **Cooperative Close**: Both parties agree, single on-chain transaction
- **Force Close**: Unilateral close with commitment transaction, subject to timelock

## Payment Features

### Multi-Path Payments (MPP)
Split large payments across multiple routes for better success rates:
```javascript
const payment = await rpc.send_payment({
  invoice: "fibt1...",
  max_parts: "0x4"  // Split into up to 4 parts
});
```

### Trampoline Routing
Simplified routing for light clients - delegate path-finding to intermediate nodes:
```javascript
const payment = await rpc.send_payment({
  invoice: "fibt1...",
  trampoline_hops: [
    { pubkey: "0x...", fee_rate: "0x3e8" }
  ]
});
```

### Keysend Payments
Send payments without pre-generated invoice:
```javascript
const payment = await rpc.send_payment({
  target_pubkey: "0x...",
  amount: "0x5f5e100",
  keysend: true
});
```

### Custom Records
Attach application-specific data to payments (up to 2KB):
```javascript
const payment = await rpc.send_payment({
  invoice: "fibt1...",
  custom_records: {
    "65536": "0x48656c6c6f"  // Custom TLV data
  }
});
```

### Hold Invoices
Create invoices where preimage is provided later:
```javascript
// Create invoice with payment_hash (no preimage)
const invoice = await rpc.new_invoice({
  amount: "0x5f5e100",
  currency: "Fibt",
  payment_hash: "0x..."  // Provide hash, not preimage
});

// Later, settle with preimage
await rpc.settle_invoice({
  payment_hash: "0x...",
  payment_preimage: "0x..."
});
```

## Cross-Chain Hub (CCH)

Bridge between Bitcoin Lightning Network and Fiber:

### Send BTC (CKB → BTC Lightning)
```javascript
// Pay a Bitcoin Lightning invoice using CKB funds
const order = await rpc.send_btc({
  btc_pay_req: "lnbc1...",  // Bitcoin Lightning invoice
  currency: "Fibt"
});
```

### Receive BTC (BTC Lightning → CKB)
```javascript
// Create order to receive BTC Lightning payment
const order = await rpc.receive_btc({
  fiber_pay_req: "fibt1..."  // Fiber invoice
});
```

## RPC API Reference

For complete API documentation, see [references/RPC_API.md](references/RPC_API.md).

### Quick Reference

**Node & Peers**:
- `node_info` - Get node information
- `connect_peer` - Connect to a peer
- `disconnect_peer` - Disconnect from a peer
- `list_peers` - List connected peers

**Channels**:
- `open_channel` - Open a new channel
- `accept_channel` - Accept incoming channel
- `list_channels` - List all channels
- `update_channel` - Update channel parameters
- `shutdown_channel` - Close a channel (cooperative or force)
- `abandon_channel` - Remove non-ready channel

**Invoices**:
- `new_invoice` - Create payment invoice
- `parse_invoice` - Parse invoice string
- `get_invoice` - Get invoice status
- `cancel_invoice` - Cancel unpaid invoice
- `settle_invoice` - Settle hold invoice with preimage

**Payments**:
- `send_payment` - Send payment (invoice, keysend, MPP, trampoline)
- `get_payment` - Get payment status
- `build_router` - Build custom route
- `send_payment_with_router` - Send with pre-built route

**Network Graph**:
- `graph_nodes` - List network nodes (paginated)
- `graph_channels` - List network channels (paginated)

**Cross-Chain Hub**:
- `send_btc` - Pay BTC Lightning invoice
- `receive_btc` - Receive BTC Lightning payment
- `get_cch_order` - Get cross-chain order status

### Common Examples

```bash
# Connect to testnet node
curl -X POST http://127.0.0.1:8227 -H "Content-Type: application/json" -d '{
  "jsonrpc": "2.0",
  "method": "connect_peer",
  "params": [{"address": "/ip4/18.162.235.225/tcp/8119/p2p/QmXen3eUHhywmutEzydCsW4hXBoeVmdET2FJvMX69XJ1Eo"}],
  "id": 1
}'

# Open CKB channel (500 CKB)
curl -X POST http://127.0.0.1:8227 -H "Content-Type: application/json" -d '{
  "jsonrpc": "2.0",
  "method": "open_channel",
  "params": [{
    "peer_id": "QmXen3eUHhywmutEzydCsW4hXBoeVmdET2FJvMX69XJ1Eo",
    "funding_amount": "0xba43b7400",
    "public": true
  }],
  "id": 1
}'

# Open UDT channel (10 RUSD)
curl -X POST http://127.0.0.1:8227 -H "Content-Type: application/json" -d '{
  "jsonrpc": "2.0",
  "method": "open_channel",
  "params": [{
    "peer_id": "QmXen3eUHhywmutEzydCsW4hXBoeVmdET2FJvMX69XJ1Eo",
    "funding_amount": "0x3b9aca00",
    "public": true,
    "funding_udt_type_script": {
      "code_hash": "0x1142755a044bf2ee358cba9f2da187ce928c91cd4dc8692ded0337efa677d21a",
      "hash_type": "type",
      "args": "0x878fcc6f1f08d48e87bb1c3b3d5083f23f8a39c5d5c764f253b55b998526439b"
    }
  }],
  "id": 1
}'

# Send payment with MPP and fee limit
curl -X POST http://127.0.0.1:8227 -H "Content-Type: application/json" -d '{
  "jsonrpc": "2.0",
  "method": "send_payment",
  "params": [{
    "invoice": "fibt1000000001p...",
    "max_parts": "0x4",
    "max_fee_amount": "0xf4240"
  }],
  "id": 1
}'

# Dry-run payment (simulate without sending)
curl -X POST http://127.0.0.1:8227 -H "Content-Type: application/json" -d '{
  "jsonrpc": "2.0",
  "method": "send_payment",
  "params": [{
    "invoice": "fibt1000000001p...",
    "dry_run": true
  }],
  "id": 1
}'
```

## Configuration

### Node Configuration (config.yml)
```yaml
fiber:
  # P2P listening address
  listening_addr: "/ip4/127.0.0.1/tcp/8228"

  # Bootstrap nodes for network discovery
  bootnode_addrs:
    - "/ip4/54.179.226.154/tcp/8228/p2p/Qmes1EBD4yNo9Ywkfe6eRw9tG1nVNGLDmMud1xJMsoYFKy"

  # Announce address to network
  announce_listening_addr: true

  # Network: mainnet, testnet, or dev
  chain: testnet

  # On-chain scripts configuration
  scripts:
    - name: FundingLock
      script:
        code_hash: 0x6c67887fe201ee0c7853f1682c0b77c0e6214044c156c7558269390a8afa6d7c
        hash_type: type
        args: 0x
      cell_deps:
        - type_id:
            code_hash: 0x00000000000000000000000000000000000000000000000000545950455f4944
            hash_type: type
            args: 0x3cb7c0304fe53f75bb5727e2484d0beae4bd99d979813c6fc97c3cca569f10f6
    - name: CommitmentLock
      script:
        code_hash: 0x740dee83f87c6f309824d8fd3fbdd3c8380ee6fc9acc90b1a748438afcdf81d8
        hash_type: type
        args: 0x
      cell_deps:
        - type_id:
            code_hash: 0x00000000000000000000000000000000000000000000000000545950455f4944
            hash_type: type
            args: 0xf7e458887495cf70dd30d1543cad47dc1dfe9d874177bf19291e4db478d5751b

rpc:
  # RPC listening address (bind to localhost for security)
  listening_addr: "127.0.0.1:8227"

ckb:
  # CKB RPC endpoint
  rpc_url: "https://testnet.ckbapp.dev/"

  # UDT whitelist for supported tokens
  udt_whitelist:
    - name: RUSD
      script:
        code_hash: 0x1142755a044bf2ee358cba9f2da187ce928c91cd4dc8692ded0337efa677d21a
        hash_type: type
        args: 0x878fcc6f1f08d48e87bb1c3b3d5083f23f8a39c5d5c764f253b55b998526439b
      cell_deps:
        - type_id:
            code_hash: 0x00000000000000000000000000000000000000000000000000545950455f4944
            hash_type: type
            args: 0x97d30b723c0b2c66e9cb8d4d0df4ab5d7222cbb00d4a9a2055ce2e5d7f0d8b0f
      auto_accept_amount: 1000000000

services:
  - fiber
  - rpc
  - ckb
```

## Important Constants

| Constant | Value | Description |
|----------|-------|-------------|
| MIN_TLC_EXPIRY_DELTA | 57,600,000 ms (16 hours) | Minimum TLC expiry delta |
| MAX_PAYMENT_TLC_EXPIRY_LIMIT | 1,209,600,000 ms (14 days) | Maximum TLC expiry limit |
| DEFAULT_TLC_EXPIRY_DELTA | 86,400,000 ms (1 day) | Default TLC expiry delta |
| DEFAULT_TLC_FEE_PROPORTIONAL_MILLIONTHS | 1000 (0.1%) | Default routing fee |
| MIN_CHANNEL_CKB_AMOUNT | 162 CKB | Minimum CKB for channel |
| RESERVED_CKB_PER_PARTY | 98 CKB | Reserved CKB per party for settlement |
| MAX_TLC_NUMBER_IN_FLIGHT | 125 | Default max concurrent TLCs |
| MAX_CUSTOM_RECORDS_SIZE | 2 KB | Max custom data per payment |
| MAX_SERVICE_PROTOCOL_DATA_SIZE | 130 KB | Max P2P message size |
| GOSSIP_SOFT_STALE | 2 weeks | Soft staleness for gossip messages |
| GOSSIP_HARD_STALE | 4 weeks | Hard staleness for gossip messages |

## Channel States

| State | Description |
|-------|-------------|
| NEGOTIATING_FUNDING | Negotiating channel parameters |
| COLLABORATING_FUNDING_TX | Building funding transaction |
| SIGNING_COMMITMENT | Exchanging commitment signatures |
| AWAITING_TX_SIGNATURES | Waiting for funding tx signatures |
| AWAITING_CHANNEL_READY | Waiting for funding tx confirmation |
| CHANNEL_READY | Channel is operational |
| SHUTTING_DOWN | Channel is being closed |
| CLOSED | Channel is closed |

## Invoice Currencies

| Currency | Network | Prefix |
|----------|---------|--------|
| Fibb | Mainnet | fibb |
| Fibt | Testnet | fibt |
| Fibd | Devnet | fibd |

## Invoice Status

| Status | Description |
|--------|-------------|
| Open | Invoice created, awaiting payment |
| Accepted | Payment received, hold invoice awaiting settlement |
| Settled | Payment settled (preimage revealed) |
| Cancelled | Invoice cancelled |

## Payment Status

| Status | Description |
|--------|-------------|
| Created | Payment initiated |
| Inflight | Payment in progress |
| Success | Payment completed |
| Failed | Payment failed |

## On-Chain Scripts

For detailed script documentation, see [references/ON_CHAIN_SCRIPTS.md](references/ON_CHAIN_SCRIPTS.md).

### Funding Lock Script
- **Purpose**: 2-of-2 MuSig2 multisig for channel funding
- **Code hash (testnet)**: `0x6c67887fe201ee0c7853f1682c0b77c0e6214044c156c7558269390a8afa6d7c`
- **Args**: 20 bytes pubkey_hash (blake160 of aggregated x-only pubkey)
- **Unlock**: Schnorr signature from aggregated key

### Commitment Lock Script
- **Purpose**: Channel state enforcement with TLC and revocation support
- **Code hash (testnet)**: `0x740dee83f87c6f309824d8fd3fbdd3c8380ee6fc9acc90b1a748438afcdf81d8`
- **Args (57 bytes)**: revocation_pubkey_hash + delay_epoch + version + settlement_script_hash + is_first_settlement

**Unlock Modes**:
1. **Revocation** (unlock_count=0x00): Punish old state broadcasts
2. **Settlement** (unlock_count>0): Normal close with TLC resolution

**TLC Settlement**:
| Type | Preimage Unlock | Expiry Unlock |
|------|-----------------|---------------|
| Offered | remote signs | local signs |
| Received | local signs | remote signs |

## Testnet Resources

### Public Nodes
```
Node 1: /ip4/18.162.235.225/tcp/8119/p2p/QmXen3eUHhywmutEzydCsW4hXBoeVmdET2FJvMX69XJ1Eo
Node 2: /ip4/18.163.221.211/tcp/8119/p2p/QmbKyzq9qUmymW2Gi8Zq7kKVpPiNA1XUJ6uMvsUC4F3p89
```

### Faucets
- CKB: https://faucet.nervos.org
- RUSD: https://testnet0815.stablepp.xyz/faucet

### RUSD Token (Testnet)
```json
{
  "code_hash": "0x1142755a044bf2ee358cba9f2da187ce928c91cd4dc8692ded0337efa677d21a",
  "hash_type": "type",
  "args": "0x878fcc6f1f08d48e87bb1c3b3d5083f23f8a39c5d5c764f253b55b998526439b"
}
```

## Common Patterns

### Pattern 1: Basic Payment Flow
```javascript
// 1. Receiver creates invoice
const invoice = await rpc.new_invoice({
  amount: "0x5f5e100", // 1 CKB
  currency: "Fibt",
  description: "Payment for goods",
  expiry: "0xe10", // 1 hour
  payment_preimage: generateRandomPreimage(),
  hash_algorithm: "sha256"
});

// 2. Sender pays invoice
const payment = await rpc.send_payment({
  invoice: invoice.invoice_address
});

// 3. Check payment status
const status = await rpc.get_payment({
  payment_hash: payment.payment_hash
});
```

### Pattern 2: Multi-hop Payment with MPP
```
Alice ──(channel)──▶ Bob ──(channel)──▶ Carol
  │                    │                   │
  │  send_payment      │                   │
  │  (invoice from     │                   │
  │   Carol, MPP)      │                   │
  │                    │                   │
  │  TLC propagates    │  TLC propagates   │
  │  (split across     │  (multiple paths) │
  │   multiple paths)  │                   │
  │                    │                   │
  │  preimage returns  │  preimage returns │
  │  ◀─────────────────│  ◀────────────────│
```

### Pattern 3: Channel Rebalancing
```javascript
// Use send_payment_with_router for circular payments
const router = await rpc.build_router({
  amount: "0x5f5e100",
  hops_info: [
    { pubkey: "0x...", channel_outpoint: "0x..." },
    { pubkey: "0x...", channel_outpoint: "0x..." },
    // ... back to self
  ]
});

await rpc.send_payment_with_router({
  router: router.router_hops,
  keysend: true,
  allow_self_payment: true
});
```

### Pattern 4: Hold Invoice for Escrow
```javascript
// 1. Create hold invoice (hash only, no preimage)
const paymentHash = sha256(preimage);
const invoice = await rpc.new_invoice({
  amount: "0x5f5e100",
  currency: "Fibt",
  payment_hash: paymentHash  // No preimage provided
});

// 2. Payer sends payment (held until settled)
await rpc.send_payment({ invoice: invoice.invoice_address });

// 3. After conditions met, settle with preimage
await rpc.settle_invoice({
  payment_hash: paymentHash,
  payment_preimage: preimage
});
```

### Pattern 5: Cross-Chain Payment (BTC Lightning)
```javascript
// Pay a Bitcoin Lightning invoice using Fiber
const order = await rpc.send_btc({
  btc_pay_req: "lnbc100n1p...",  // Bitcoin Lightning invoice
  currency: "Fibt"
});

// Check order status
const status = await rpc.get_cch_order({
  payment_hash: order.payment_hash
});
```

## Error Handling

### Common Errors
| Error | Cause | Solution |
|-------|-------|----------|
| "Failed to build route" | No path to destination | Wait for channel to be ready, or open direct channel |
| "Insufficient balance" | Not enough funds in channel | Add more funds or use different channel |
| "Channel not ready" | Channel still confirming | Wait for CHANNEL_READY state |
| "Invoice expired" | Invoice past expiry time | Generate new invoice |
| "TLC timeout" | Payment took too long | Retry with longer timeout |
| "Max TLCs in flight" | Too many pending payments | Wait for pending payments to complete |

## Security Considerations

1. **Private Key Security**: Store keys securely, use `FIBER_SECRET_KEY_PASSWORD` environment variable
2. **RPC Security**: Bind RPC to localhost only, use Biscuit authentication in production
3. **Channel Monitoring**: Run watchtower service for force-close protection
4. **Backup**: Regularly backup channel state database
5. **MuSig2**: Channel commitments use MuSig2 for aggregated signatures

## Technology Stack

| Component | Technology |
|-----------|------------|
| Language | Rust |
| P2P Framework | Tentacle |
| Actor Model | Ractor |
| RPC Framework | jsonrpsee (HTTP/WebSocket) |
| Cryptography | secp256k1, MuSig2, Schnorr |
| Storage | RocksDB |
| Onion Routing | fiber-sphinx |

## References

- [Fiber Documentation](https://docs.fiber.world/)
- [Fiber GitHub Repository](https://github.com/nervosnetwork/fiber)
- [Fiber Scripts Repository](https://github.com/nervosnetwork/fiber-scripts)
- [CKB Documentation](https://docs.nervos.org/)
- [Lightning Network BOLT Specifications](https://github.com/lightning/bolts)
