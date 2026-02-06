# Fiber RPC API Reference

Complete reference for Fiber Node JSON-RPC API.

## Connection Details

- **Default Port**: 8227
- **Protocol**: JSON-RPC 2.0
- **Content-Type**: application/json
- **Authentication**: Optional Biscuit-based authentication

## Node Information

### node_info

Get information about the local node.

**Parameters**: None

**Response**:
```json
{
  "version": "0.1.0",
  "commit_hash": "abc123...",
  "node_id": "0x...",
  "node_name": "my-fiber-node",
  "peer_id": "QmXen3eUHhywmutEzydCsW4hXBoeVmdET2FJvMX69XJ1Eo",
  "addresses": ["/ip4/127.0.0.1/tcp/8228"],
  "chain_hash": "0x...",
  "open_channel_auto_accept_min_ckb_funding_amount": "0x...",
  "auto_accept_channel_ckb_funding_amount": "0x...",
  "default_funding_lock_script": {...},
  "tlc_expiry_delta": "0x5265c00",
  "tlc_min_value": "0x0",
  "tlc_fee_proportional_millionths": "0x3e8",
  "channel_count": 5,
  "pending_channel_count": 1,
  "peers_count": 3,
  "udt_cfg_infos": [...]
}
```

## Peer Management

### connect_peer

Connect to a remote peer.

**Parameters**:
```json
{
  "address": "/ip4/18.162.235.225/tcp/8119/p2p/QmXen3eUHhywmutEzydCsW4hXBoeVmdET2FJvMX69XJ1Eo",
  "save": true  // Optional: save peer for reconnection
}
```

**Response**: `null` on success

### disconnect_peer

Disconnect from a peer.

**Parameters**:
```json
{
  "peer_id": "QmXen3eUHhywmutEzydCsW4hXBoeVmdET2FJvMX69XJ1Eo"
}
```

**Response**: `null` on success

### list_peers

List all connected peers.

**Parameters**: None or `{}`

**Response**:
```json
{
  "peers": [
    {
      "peer_id": "QmXen3eUHhywmutEzydCsW4hXBoeVmdET2FJvMX69XJ1Eo",
      "addresses": ["/ip4/18.162.235.225/tcp/8119"],
      "connected_at": "0x18a1b2c3d4e5"
    }
  ]
}
```

## Channel Management

### open_channel

Open a new payment channel with a peer.

**Parameters**:
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `peer_id` | string | Yes | Peer ID (must be connected first) |
| `funding_amount` | hex u128 | Yes | Amount to fund (shannons for CKB, smallest unit for UDT) |
| `public` | bool | No | Broadcast to network (default: true) |
| `one_way` | bool | No | One-way channel (default: false) |
| `funding_udt_type_script` | Script | No | UDT type script for token channels |
| `shutdown_script` | Script | No | Script to receive funds on close |
| `commitment_delay_epoch` | EpochNumberWithFraction | No | Delay for commitment (default: 1 epoch = 4 hours) |
| `commitment_fee_rate` | hex u64 | No | Fee rate for commitment tx |
| `funding_fee_rate` | hex u64 | No | Fee rate for funding tx |
| `tlc_expiry_delta` | hex u64 | No | TLC expiry delta in ms (default: 4 hours) |
| `tlc_min_value` | hex u128 | No | Minimum TLC value (default: 0) |
| `tlc_fee_proportional_millionths` | hex u128 | No | Routing fee in millionths (default: 1000 = 0.1%) |
| `max_tlc_value_in_flight` | hex u128 | No | Max total TLC value |
| `max_tlc_number_in_flight` | hex u64 | No | Max concurrent TLCs (default: 125) |

**Response**:
```json
{
  "temporary_channel_id": "0x..."
}
```

**Example - CKB Channel (500 CKB)**:
```bash
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
```

**Example - UDT Channel (10 RUSD)**:
```bash
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
```

### accept_channel

Accept an incoming channel request.

**Parameters**:
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `temporary_channel_id` | Hash256 | Yes | Temporary channel ID from open request |
| `funding_amount` | hex u128 | Yes | Amount to contribute |
| `shutdown_script` | Script | No | Script to receive funds on close |
| `max_tlc_value_in_flight` | hex u128 | No | Max total TLC value (default: u128::MAX) |
| `max_tlc_number_in_flight` | hex u64 | No | Max concurrent TLCs (default: 125) |
| `tlc_min_value` | hex u128 | No | Minimum TLC value |
| `tlc_fee_proportional_millionths` | hex u128 | No | Routing fee |
| `tlc_expiry_delta` | hex u64 | No | TLC expiry delta |

**Response**:
```json
{
  "channel_id": "0x..."  // Final channel ID
}
```

### list_channels

List channels, optionally filtered by peer.

**Parameters**:
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `peer_id` | string | No | Filter by peer ID |
| `include_closed` | bool | No | Include closed channels (default: false) |

**Response**:
```json
{
  "channels": [
    {
      "channel_id": "0x...",
      "is_public": true,
      "is_acceptor": false,
      "is_one_way": false,
      "channel_outpoint": "0x...",
      "peer_id": "Qm...",
      "funding_udt_type_script": null,
      "state": {
        "state_name": "CHANNEL_READY",
        "state_flags": null
      },
      "local_balance": "0x...",
      "offered_tlc_balance": "0x0",
      "remote_balance": "0x...",
      "received_tlc_balance": "0x0",
      "pending_tlcs": [],
      "latest_commitment_transaction_hash": "0x...",
      "created_at": "0x...",
      "enabled": true,
      "tlc_expiry_delta": "0x5265c00",
      "tlc_fee_proportional_millionths": "0x3e8",
      "shutdown_transaction_hash": null
    }
  ]
}
```

### update_channel

Update channel parameters.

**Parameters**:
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `channel_id` | Hash256 | Yes | Channel ID |
| `enabled` | bool | No | Enable/disable channel |
| `tlc_expiry_delta` | hex u64 | No | New TLC expiry delta |
| `tlc_minimum_value` | hex u128 | No | New minimum TLC value |
| `tlc_fee_proportional_millionths` | hex u128 | No | New routing fee |

**Response**: `null` on success

### shutdown_channel

Close a channel (cooperative or force).

**Parameters**:
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `channel_id` | Hash256 | Yes | Channel ID |
| `close_script` | Script | No | Script to receive funds (required if not force) |
| `fee_rate` | hex u64 | No | Fee rate for closing tx (default: 1000 shannons/KW) |
| `force` | bool | No | Force close (default: false) |

**Response**: `null` on success

**Example - Cooperative Close**:
```bash
curl -X POST http://127.0.0.1:8227 -H "Content-Type: application/json" -d '{
  "jsonrpc": "2.0",
  "method": "shutdown_channel",
  "params": [{
    "channel_id": "0x...",
    "close_script": {
      "code_hash": "0x9bd7e06f3ecf4be0f2fcd2188b23f1b9fcc88e5d4b65a8637b17723bbda3cce8",
      "hash_type": "type",
      "args": "0x..."
    },
    "fee_rate": "0x3FC"
  }],
  "id": 1
}'
```

**Example - Force Close**:
```bash
curl -X POST http://127.0.0.1:8227 -H "Content-Type: application/json" -d '{
  "jsonrpc": "2.0",
  "method": "shutdown_channel",
  "params": [{
    "channel_id": "0x...",
    "force": true
  }],
  "id": 1
}'
```

### abandon_channel

Remove a channel from the node (only for non-ready/non-closed channels).

**Parameters**:
```json
{
  "channel_id": "0x..."
}
```

**Response**: `null` on success

## Invoice Management

### new_invoice

Create a new payment invoice.

**Parameters**:
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `amount` | hex u128 | Yes | Payment amount |
| `currency` | string | Yes | "Fibb" (mainnet), "Fibt" (testnet), "Fibd" (devnet) |
| `description` | string | No | Human-readable description |
| `expiry` | hex u64 | No | Expiry time in seconds (default: 3600) |
| `payment_preimage` | Hash256 | No* | 32-byte preimage for payment hash |
| `payment_hash` | Hash256 | No* | 32-byte payment hash (for hold invoices) |
| `hash_algorithm` | string | No | "sha256" or "blake2b" (default: "sha256") |
| `udt_type_script` | Script | No | UDT type script for token invoices |
| `final_expiry_delta` | hex u64 | No | Final TLC expiry delta |
| `fallback_address` | string | No | Fallback on-chain address |
| `allow_mpp` | bool | No | Allow multi-path payments (default: true) |
| `allow_trampoline_routing` | bool | No | Allow trampoline routing (default: false) |

*Either `payment_preimage` or `payment_hash` required. Use `payment_hash` for hold invoices.

**Response**:
```json
{
  "invoice_address": "fibt1000000001p...",
  "invoice": {
    "currency": "Fibt",
    "amount": "0x5f5e100",
    "signature": "0x...",
    "data": {
      "timestamp": "0x...",
      "payment_hash": "0x...",
      "attrs": [...]
    }
  }
}
```

### parse_invoice

Parse and validate an invoice string.

**Parameters**:
```json
{
  "invoice": "fibt1000000001p..."
}
```

**Response**:
```json
{
  "invoice": {
    "currency": "Fibt",
    "amount": "0x5f5e100",
    "signature": "0x...",
    "data": {
      "timestamp": "0x...",
      "payment_hash": "0x...",
      "attrs": [...]
    }
  }
}
```

### get_invoice

Get invoice status by payment hash.

**Parameters**:
```json
{
  "payment_hash": "0x..."
}
```

**Response**:
```json
{
  "invoice_address": "fibt1...",
  "invoice": {...},
  "status": "Paid"  // "Open", "Accepted", "Settled", "Cancelled"
}
```

### cancel_invoice

Cancel an unpaid invoice.

**Parameters**:
```json
{
  "payment_hash": "0x..."
}
```

**Response**: Invoice details on success

### settle_invoice

Settle a hold invoice by providing the preimage.

**Parameters**:
```json
{
  "payment_hash": "0x...",
  "payment_preimage": "0x..."
}
```

**Response**: `null` on success

**Example - Hold Invoice Flow**:
```bash
# 1. Create hold invoice (with payment_hash, not preimage)
curl -X POST http://127.0.0.1:8227 -H "Content-Type: application/json" -d '{
  "jsonrpc": "2.0",
  "method": "new_invoice",
  "params": [{
    "amount": "0x5f5e100",
    "currency": "Fibt",
    "payment_hash": "0x..."
  }],
  "id": 1
}'

# 2. Later, settle with preimage
curl -X POST http://127.0.0.1:8227 -H "Content-Type: application/json" -d '{
  "jsonrpc": "2.0",
  "method": "settle_invoice",
  "params": [{
    "payment_hash": "0x...",
    "payment_preimage": "0x..."
  }],
  "id": 1
}'
```

## Payment Operations

### send_payment

Send a payment using an invoice, keysend, or with advanced options.

**Parameters**:
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `invoice` | string | No* | Invoice string to pay |
| `target_pubkey` | hex | No* | Target node public key (for keysend) |
| `amount` | hex u128 | No* | Payment amount (for keysend or overpayment) |
| `payment_hash` | Hash256 | No | Payment hash (for keysend without invoice) |
| `keysend` | bool | No | Enable keysend mode (default: false) |
| `udt_type_script` | Script | No | UDT type script (for keysend) |
| `timeout` | hex u64 | No | Payment timeout in seconds |
| `max_fee_amount` | hex u128 | No | Maximum routing fee |
| `max_fee_rate` | hex u64 | No | Maximum fee rate (millionths) |
| `max_parts` | hex u64 | No | Maximum payment parts for MPP |
| `final_tlc_expiry_delta` | hex u64 | No | Final TLC expiry delta |
| `tlc_expiry_limit` | hex u64 | No | Maximum TLC expiry |
| `hop_hints` | array | No | Routing hints for private channels |
| `trampoline_hops` | array | No | Trampoline routing hops |
| `allow_self_payment` | bool | No | Allow payment to self (default: false) |
| `custom_records` | object | No | Custom TLV records (max 2KB total) |
| `dry_run` | bool | No | Simulate without sending (default: false) |

*Either `invoice` or (`target_pubkey` + `amount` + `keysend`) required

**Response**:
```json
{
  "payment_hash": "0x...",
  "status": "Inflight",  // "Created", "Inflight", "Success", "Failed"
  "created_at": "0x...",
  "last_updated_at": "0x...",
  "failed_error": null,
  "fee": "0x...",
  "custom_records": {...},
  "routers": [...]
}
```

**Example - Invoice Payment**:
```bash
curl -X POST http://127.0.0.1:8227 -H "Content-Type: application/json" -d '{
  "jsonrpc": "2.0",
  "method": "send_payment",
  "params": [{
    "invoice": "fibt1000000001p..."
  }],
  "id": 1
}'
```

**Example - Keysend Payment**:
```bash
curl -X POST http://127.0.0.1:8227 -H "Content-Type: application/json" -d '{
  "jsonrpc": "2.0",
  "method": "send_payment",
  "params": [{
    "target_pubkey": "0x...",
    "amount": "0x5f5e100",
    "keysend": true
  }],
  "id": 1
}'
```

**Example - Multi-Path Payment with Fee Limit**:
```bash
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
```

**Example - Payment with Custom Records**:
```bash
curl -X POST http://127.0.0.1:8227 -H "Content-Type: application/json" -d '{
  "jsonrpc": "2.0",
  "method": "send_payment",
  "params": [{
    "invoice": "fibt1000000001p...",
    "custom_records": {
      "65536": "0x48656c6c6f"
    }
  }],
  "id": 1
}'
```

**Example - Dry Run (Simulate)**:
```bash
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

**Example - Trampoline Routing**:
```bash
curl -X POST http://127.0.0.1:8227 -H "Content-Type: application/json" -d '{
  "jsonrpc": "2.0",
  "method": "send_payment",
  "params": [{
    "invoice": "fibt1000000001p...",
    "trampoline_hops": [
      {"pubkey": "0x...", "fee_rate": "0x3e8"}
    ]
  }],
  "id": 1
}'
```

### get_payment

Get payment status by payment hash.

**Parameters**:
```json
{
  "payment_hash": "0x..."
}
```

**Response**:
```json
{
  "payment_hash": "0x...",
  "status": "Success",
  "created_at": "0x...",
  "last_updated_at": "0x...",
  "failed_error": null,
  "fee": "0x3e8",
  "custom_records": {...},
  "routers": [...]
}
```

### build_router

Build a custom payment route.

**Parameters**:
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `amount` | hex u128 | No | Payment amount |
| `udt_type_script` | Script | No | UDT type script |
| `hops_info` | array | Yes | Array of hop information |
| `final_tlc_expiry_delta` | hex u64 | No | Final TLC expiry delta |

**Hop Info**:
```json
{
  "pubkey": "0x...",
  "channel_outpoint": "0x..."
}
```

**Response**:
```json
{
  "router_hops": [...]
}
```

### send_payment_with_router

Send payment using a pre-built route.

**Parameters**:
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `payment_hash` | Hash256 | No | Payment hash |
| `router` | array | Yes | Router hops from build_router |
| `invoice` | string | No | Invoice to pay |
| `keysend` | bool | No | Keysend mode |
| `udt_type_script` | Script | No | UDT type script |
| `custom_records` | object | No | Custom TLV records |
| `dry_run` | bool | No | Simulate only |

**Response**: Same as `send_payment`

## Network Graph

### graph_nodes

List nodes in the network graph with pagination.

**Parameters**:
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `limit` | hex u64 | No | Maximum nodes to return |
| `after` | string | No | Cursor for pagination |

**Response**:
```json
{
  "nodes": [
    {
      "node_id": "0x...",
      "timestamp": "0x...",
      "version": "0.1.0",
      "features": {...},
      "addresses": ["/ip4/..."],
      "auto_accept_min_ckb_funding_amount": "0x..."
    }
  ],
  "last_cursor": "0x..."
}
```

### graph_channels

List channels in the network graph with pagination.

**Parameters**:
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `limit` | hex u64 | No | Maximum channels to return |
| `after` | string | No | Cursor for pagination |

**Response**:
```json
{
  "channels": [
    {
      "channel_outpoint": "0x...",
      "timestamp": "0x...",
      "capacity": "0x...",
      "node1": "0x...",
      "node2": "0x...",
      "update_of_node1": {...},
      "update_of_node2": {...}
    }
  ],
  "last_cursor": "0x..."
}
```

## Cross-Chain Hub (CCH)

The Cross-Chain Hub enables interoperability between Fiber (CKB) and Bitcoin Lightning Network.

### send_btc

Pay a Bitcoin Lightning invoice using CKB funds.

**Parameters**:
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `btc_pay_req` | string | Yes | Bitcoin Lightning invoice (BOLT11) |
| `currency` | string | Yes | Currency ("Fibt" for testnet) |

**Response**:
```json
{
  "payment_hash": "0x...",
  "status": "Pending",
  "created_at": "0x..."
}
```

**Example**:
```bash
curl -X POST http://127.0.0.1:8227 -H "Content-Type: application/json" -d '{
  "jsonrpc": "2.0",
  "method": "send_btc",
  "params": [{
    "btc_pay_req": "lnbc100n1p...",
    "currency": "Fibt"
  }],
  "id": 1
}'
```

### receive_btc

Create an order to receive a Bitcoin Lightning payment on Fiber.

**Parameters**:
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `fiber_pay_req` | string | Yes | Fiber invoice to receive payment |

**Response**:
```json
{
  "payment_hash": "0x...",
  "btc_pay_req": "lnbc...",
  "status": "Pending",
  "created_at": "0x..."
}
```

**Example**:
```bash
curl -X POST http://127.0.0.1:8227 -H "Content-Type: application/json" -d '{
  "jsonrpc": "2.0",
  "method": "receive_btc",
  "params": [{
    "fiber_pay_req": "fibt1..."
  }],
  "id": 1
}'
```

### get_cch_order

Get the status of a cross-chain hub order.

**Parameters**:
```json
{
  "payment_hash": "0x..."
}
```

**Response**:
```json
{
  "payment_hash": "0x...",
  "status": "Completed",
  "created_at": "0x...",
  "updated_at": "0x..."
}
```

## Watchtower

The watchtower service monitors channels for breach attempts and can submit penalty transactions.

### create_watch_channel

Register a channel for watchtower monitoring.

**Parameters**:
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `channel_id` | Hash256 | Yes | Channel ID to watch |
| `funding_tx_hash` | Hash256 | Yes | Funding transaction hash |
| `funding_tx_index` | hex u32 | Yes | Funding output index |

**Response**: Watch channel details

### remove_watch_channel

Stop watching a channel.

**Parameters**:
```json
{
  "channel_id": "0x..."
}
```

**Response**: `null` on success

### update_revocation

Update revocation data for a watched channel.

**Parameters**:
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `channel_id` | Hash256 | Yes | Channel ID |
| `revocation_data` | object | Yes | Revocation secrets and data |

**Response**: `null` on success

### update_pending_remote_settlement

Update pending remote settlement data.

**Parameters**: Settlement data object

**Response**: `null` on success

### update_local_settlement

Update local settlement data.

**Parameters**: Settlement data object

**Response**: `null` on success

### create_preimage

Store a preimage for watchtower use.

**Parameters**:
```json
{
  "payment_hash": "0x...",
  "preimage": "0x..."
}
```

**Response**: `null` on success

### remove_preimage

Remove a stored preimage.

**Parameters**:
```json
{
  "payment_hash": "0x..."
}
```

**Response**: `null` on success

## Development APIs (Debug Builds Only)

These APIs are only available in debug builds for testing purposes.

### commitment_signed

Manually trigger commitment signing.

**Parameters**:
```json
{
  "channel_id": "0x..."
}
```

### add_tlc

Manually add a TLC to a channel.

**Parameters**:
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `channel_id` | Hash256 | Yes | Channel ID |
| `amount` | hex u128 | Yes | TLC amount |
| `payment_hash` | Hash256 | Yes | Payment hash |
| `expiry` | hex u64 | Yes | TLC expiry |
| `hash_algorithm` | string | No | Hash algorithm |

**Response**:
```json
{
  "tlc_id": "0x..."
}
```

### remove_tlc

Manually remove a TLC from a channel.

**Parameters**:
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `channel_id` | Hash256 | Yes | Channel ID |
| `tlc_id` | hex u64 | Yes | TLC ID |
| `reason` | object | Yes | Removal reason (fulfill or fail) |

### submit_commitment_transaction

Submit a commitment transaction on-chain.

**Parameters**:
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `channel_id` | Hash256 | Yes | Channel ID |
| `commitment_number` | hex u64 | Yes | Commitment number |

**Response**:
```json
{
  "tx_hash": "0x..."
}
```

### check_channel_shutdown

Check if a channel can be shut down.

**Parameters**:
```json
{
  "channel_id": "0x..."
}
```

## Channel States

| State | Description |
|-------|-------------|
| `NEGOTIATING_FUNDING` | Negotiating channel parameters |
| `COLLABORATING_FUNDING_TX` | Building funding transaction |
| `SIGNING_COMMITMENT` | Exchanging commitment signatures |
| `AWAITING_TX_SIGNATURES` | Waiting for funding tx signatures |
| `AWAITING_CHANNEL_READY` | Waiting for funding tx confirmation |
| `CHANNEL_READY` | Channel is operational |
| `SHUTTING_DOWN` | Channel is being closed |
| `CLOSED` | Channel is closed |

## Payment Status

| Status | Description |
|--------|-------------|
| `Created` | Payment initiated |
| `Inflight` | Payment in progress |
| `Success` | Payment completed successfully |
| `Failed` | Payment failed |

## Invoice Status

| Status | Description |
|--------|-------------|
| `Open` | Invoice created, awaiting payment |
| `Accepted` | Payment received, hold invoice awaiting settlement |
| `Settled` | Payment settled (preimage revealed) |
| `Cancelled` | Invoice cancelled |

## Error Handling

Errors follow JSON-RPC 2.0 format:
```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32000,
    "message": "Error description",
    "data": {...}
  },
  "id": 1
}
```

Common error codes:
- `-32600`: Invalid request
- `-32601`: Method not found
- `-32602`: Invalid params
- `-32603`: Internal error
- `-32000`: Execution failed (custom)

## API Summary by Module

| Module | Methods |
|--------|---------|
| **Info** | `node_info` |
| **Peer** | `connect_peer`, `disconnect_peer`, `list_peers` |
| **Channel** | `open_channel`, `accept_channel`, `list_channels`, `update_channel`, `shutdown_channel`, `abandon_channel` |
| **Invoice** | `new_invoice`, `parse_invoice`, `get_invoice`, `cancel_invoice`, `settle_invoice` |
| **Payment** | `send_payment`, `get_payment`, `build_router`, `send_payment_with_router` |
| **Graph** | `graph_nodes`, `graph_channels` |
| **CCH** | `send_btc`, `receive_btc`, `get_cch_order` |
| **Watchtower** | `create_watch_channel`, `remove_watch_channel`, `update_revocation`, `update_pending_remote_settlement`, `update_local_settlement`, `create_preimage`, `remove_preimage` |
| **Dev** | `commitment_signed`, `add_tlc`, `remove_tlc`, `submit_commitment_transaction`, `check_channel_shutdown` |
