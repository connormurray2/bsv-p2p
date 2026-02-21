# Section 6: Security Considerations

## 6.1 Threat Model

Payment channels involve locked funds and off-chain state updates. We assume:

- **Network adversary**: Can delay, reorder, or drop messages (but not break TLS/noise encryption)
- **Byzantine counterparty**: May attempt double-spend, publish stale states, or go offline
- **Malicious third parties**: May attempt to impersonate peers or inject false messages

### In-Scope Threats

| Threat | Mitigation |
|--------|------------|
| Counterparty publishes old state | nSequence ordering ensures latest state wins in mempool |
| Counterparty goes offline | nLockTime force-close after 144 blocks (~24h) |
| Message replay | Channel ID + sequence number prevent replay |
| Peer impersonation | libp2p cryptographic peer IDs (Ed25519) |
| Man-in-the-middle | Noise protocol encryption on all connections |

### Out-of-Scope

- Endpoint compromise (key theft from peer's machine)
- Eclipse attacks on Bitcoin network
- 51% attacks on BSV chain

## 6.2 Cryptographic Requirements

### Key Management

- **Identity keys**: Ed25519 for libp2p peer identity
- **Channel keys**: secp256k1 for Bitcoin signatures
- **Key derivation**: Channels SHOULD use derived keys per channel (BIP-32 hardened derivation)

### Signature Validation

All commitment transactions MUST be validated:

```
1. Verify SIGHASH_ALL signature from counterparty
2. Verify correct output amounts (sum = channel capacity)
3. Verify nSequence > previous commitment
4. Verify scriptPubKey matches agreed addresses
```

Implementations MUST reject any commitment where signature verification fails.

## 6.3 State Integrity

### Sequence Number Invariants

- Initial funding: `nSequence = 0xFFFFFFFE` (144-block relative locktime)
- Each update: `nSequence` MUST increment
- Implementations MUST persist highest-seen sequence before ACKing

### Atomicity Requirements

State updates are atomic:
1. Receive counterparty's signed commitment
2. Validate signature and amounts
3. Persist to durable storage
4. Send own signed commitment
5. Persist ACK

**Critical**: Steps 3 and 5 MUST be durable writes. Crash between 4-5 requires recovery protocol.

## 6.4 Replay Protection

Each message includes:
- `channelId`: 32-byte unique identifier
- `sequence`: Monotonically increasing counter
- `timestamp`: Wall-clock time (advisory, not trusted)

Implementations MUST track highest-seen sequence per channel and reject:
- Duplicate sequence numbers
- Sequence numbers ≤ highest-seen
- Messages for unknown channel IDs

## 6.5 Force Close Security

When counterparty is unresponsive:

1. Broadcast latest commitment transaction
2. Wait 144 blocks (relative timelock)
3. Sweep output to own address

### Watching for Fraud

Implementations SHOULD monitor the mempool and chain for:
- Counterparty's commitment transactions
- Ensure only the latest state is confirmed

If stale state detected, broadcast correct state immediately — nSequence ordering means higher sequence wins mempool priority.

## 6.6 Peer Identity Verification

### Current Model (v1)

libp2p PeerIDs are cryptographically bound to Ed25519 keys. This provides:
- **Authentication**: Messages are signed by the peer's key
- **Integrity**: Noise protocol prevents tampering

### Known Limitation

PeerID alone doesn't prove *which agent* you're talking to. An attacker could:
1. Generate new PeerID
2. Claim to be "AgentX"
3. Request channel open

### Recommended Mitigations

- **Out-of-band verification**: Exchange PeerIDs through trusted channels
- **Reputation systems**: Track behavior per PeerID over time
- **BRC-42/43 identity binding**: Future work to link PeerIDs to Paymail or other identity systems

## 6.7 Denial of Service

### Channel Spam

Malicious peers could:
- Open many channels without funding
- Send rapid UPDATE messages

**Mitigations**:
- Rate limit PROPOSE messages per peer (recommend: 1/minute)
- Require FUNDED confirmation before accepting UPDATEs
- Timeout pending channels after 10 minutes

### Message Flooding

- Implementations SHOULD enforce 64KB message limit
- Implementations SHOULD rate limit messages (recommend: 10/second/peer)
- Peers exceeding limits MAY be disconnected

## 6.8 Privacy Considerations

### On-Chain Privacy

- Funding and settlement transactions are public
- Channel capacity is visible
- Use separate addresses per channel

### Off-Chain Privacy

- UPDATE messages are peer-to-peer only
- Payment amounts visible only to channel parties
- Message content (service requests) visible only to parties

### Metadata Leakage

- libp2p connections reveal IP addresses
- Consider Tor/I2P transport for sensitive deployments
- PeerID is persistent — consider rotation for unlinkability

---

*Security section authored by Moneo 🪙*
*Review and hardening by Ghanima 🐆 welcome*
