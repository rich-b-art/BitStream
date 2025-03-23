# BitStream: Stacks Layer 2 Payment Channel Protocol

Trust-minimized payment channels for high-throughput microtransactions on Stacks (Bitcoin L2) with Bitcoin-finalized dispute resolution.

## Overview

BitStream implements scalable off-chain payment channels anchored to Bitcoin's security model. It enables instant STX-denominated transactions between parties with on-chain settlement guarantees. Channels support dynamic funding, bidirectional payments, and offer two closure modes: cooperative multisignature closures and Bitcoin-timestamped dispute resolutions.

## Key Features

- **Bitcoin-Finalized Disputes**: Challenge periods use Bitcoin's block time via Stacks epoch tracking
- **Dynamic Channel Funding**: Participants can add funds to open channels
- **Optimistic Settlements**: Unilateral closures with 1008-block (≈1 week) dispute windows
- **Replay Protection**: State updates guarded by monotonically increasing nonces
- **Trust-Minimized Design**: Clear exit pathways with enforceable on-chain settlement
- **Emergency Recovery**: Contract owner withdrawal failsafe (controlled decentralization)

## Technical Specification

### Data Model

#### `payment-channels` Map

```clarity
{
  channel-id: (buff 32),       // Unique channel ID (SHA256 hash of init params)
  participant-a: principal,     // Channel initiator's Stacks address
  participant-b: principal      // Counterparty's Stacks address
}
->
{
  total-deposited: uint,        // Total STX locked in channel
  balance-a: uint,             // Participant A's current balance
  balance-b: uint,             // Participant B's current balance
  is-open: bool,               // Channel status (true = operational)
  dispute-deadline: uint,      // Block height for dispute expiration
  nonce: uint                  // State version counter
}
```

### Constants

- `CONTRACT-OWNER`: Deployment principal with emergency withdrawal rights
- Error codes (full list in [Error Handling](#error-handling) section)

## Workflow

1. **Channel Creation**
   - Participant A initializes channel with STX deposit
   - Channel ID generated from init parameters
2. **Funding**
   - Either party can add funds while channel is open
3. **Off-Chain Updates**
   - Participants exchange signed balance states
   - Nonce increments with each state update
4. **Closure**
   - **Cooperative**: Mutual signature closure with instant settlement
   - **Unilateral**: Single-party closure with 1008-block dispute period
5. **Dispute Resolution**
   - Counterparties can submit newer state during challenge window
   - Final settlement occurs after dispute period expiration

## Core Functions

### `create-channel`

Initialize new payment channel between two parties

```clarity
(create-channel
  (channel-id (buff 32))
  (participant-b principal)
  (initial-deposit uint)
)
```

- **Parameters**:
  - `channel-id`: 32-byte channel identifier
  - `participant-b`: Counterparty address
  - `initial-deposit`: STX amount from initiator
- **Effects**:
  - Transfers STX to contract escrow
  - Creates channel record with initial balances

### `fund-channel`

Add funds to existing open channel

```clarity
(fund-channel
  (channel-id (buff 32))
  (participant-b principal)
  (additional-funds uint)
)
```

- **Requirements**:
  - Caller must be channel participant
  - Channel in open state

### `close-channel-cooperative`

Mutually-signed instant closure

```clarity
(close-channel-cooperative
  (channel-id (buff 32))
  (participant-b principal)
  (balance-a uint)
  (balance-b uint)
  (signature-a (buff 65))
  (signature-b (buff 65))
)
```

- **Verifications**:
  - Both participants' signatures on final state
  - ∑ balances = total deposited STX

### `initiate-unilateral-close`

Start disputed closure process

```clarity
(initiate-unilateral-close
  (channel-id (buff 32))
  (participant-b principal)
  (proposed-balance-a uint)
  (proposed-balance-b uint)
  (signature (buff 65))
)
```

- **Effects**:
  - Freezes channel state
  - Sets dispute deadline to `stacks-block-height + 1008`

### `resolve-unilateral-close`

Finalize after dispute period

```clarity
(resolve-unilateral-close
  (channel-id (buff 32))
  (participant-b principal)
)
```

- **Requirements**:
  - Dispute period expired
  - No newer state submitted

## Security Model

### Signature Verification

- ECDSA secp256k1 signatures (Bitcoin-compatible)
- Message format: `channel-id || balance-a || balance-b`
- **Note**: Current implementation uses simplified validation for testing

### Dispute Safeguards

- 1008-block delay (~1 week) for unilateral closures
- State nonces prevent replay attacks
- On-chain balance verification during settlements

## Error Handling

| Code   | Description                   |
| ------ | ----------------------------- |
| `u100` | Unauthorized operation        |
| `u101` | Channel already exists        |
| `u102` | Channel not found             |
| `u103` | Invalid balance distribution  |
| `u104` | Signature verification failed |
| `u105` | Channel not in open state     |
| `u106` | Dispute period still active   |
| `u107` | Invalid input parameters      |

## Emergency Procedures

### `emergency-withdraw`

```clarity
(define-public (emergency-withdraw)
```

- **Access**: Contract owner only
- **Function**: Withdraws all contract-held STX
- **Purpose**: Recovery mechanism for protocol failures

**WARNING**: Centralized failsafe - users must trust owner not to activate maliciously

## Usage Example

### Creating and Closing a Channel

1. **Creation**

```clarity
(create-channel 0xdeafbeef... sender2 500000)
```

2. **Funding**

```clarity
(fund-channel 0xdeafbeef... sender2 250000)
```

3. **Cooperative Closure**

```clarity
(close-channel-cooperative
  0xdeafbeef...
  sender2
  600000
  150000
  sigA
  sigB
)
```

## Audit Considerations

1. Signature verification implementation (currently simplified)
2. Dispute period duration validation
3. STX escrow management correctness
4. Replay protection via nonce increments
5. Emergency function governance model
