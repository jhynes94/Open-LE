# Open-LE
Open Low Energy - An open source Pay-to-Decrypt Network
# Pay-to-Decrypt (OLE Protocol) — v1.1 Specification

## 1. Overview
The Pay-to-Decrypt (OLE Protocol) enables low-power Bluetooth devices to offload stored data through third-party relays (e.g., smartphones) to their owners, while ensuring payment to the relay provider. The system operates on a blockchain network using smart contracts for decentralized settlement, data authenticity, and delivery verification.

Version **1.1** introduces a **two-phase settlement mechanism** where payments are released only after the originating device receives and acknowledges an ACK challenge.

---

## 2. Entities
- **Device**: A low-power Bluetooth-enabled sensor that collects and stores data.
- **Relay**: A mobile or fixed node capable of receiving data from a Device and forwarding it to the blockchain.
- **Blockchain**: The decentralized settlement and storage coordination layer.
- **Company**: The data owner who can decrypt received payloads.

---

## 3. Security Principles
1. **Data Encryption**: Device → Company only, AES-256 or equivalent. Keys provisioned at manufacturing.
2. **Payload Signing**: Device signs payload metadata with manufacturing key.
3. **ACK Authenticity**: ACK is generated on-chain and must be signed by the Device before payment release.
4. **Replay Prevention**: Payloads include a nonce; duplicate submissions are rejected.
5. **Proof-of-Delivery Token**: On-chain marker proving successful end-to-end delivery.

---

## 4. Protocol Flow (Two-Phase Settlement)

### Phase 1 — Data Upload (Pending)
1. **BLE Discovery**  
   Device → Relay: BLE Advertise (`HasData=1`, `RequiresACK=1`).
2. **Metadata Request**  
   Relay → Device: `REQ_META`.
3. **Metadata Response**  
   Device → Relay: `{DeviceID, Nonce, Hash, Size}` (signed).
4. **Payload Request**  
   Relay → Device: `REQ_DATA`.
5. **Payload Transfer**  
   Device → Relay: Encrypted data over BLE GATT chunks.
6. **Submit Payload**  
   Relay → Blockchain: `submitPayload(DeviceID, Nonce, Hash, Payload)`.
7. **ACK Challenge Issuance**  
   Blockchain → Relay: `AckChallengeToken` (contract-signed, Payment=Pending).

### Phase 2 — ACK Delivery
8. **Deliver ACK Challenge**  
   Relay → Device: `AckChallengeToken`.
9. **Device ACK Signature**  
   Device → Relay: `Sign(AckChallengeToken)` with manufacturing key.
10. **Confirm ACK**  
    Relay → Blockchain: `confirmAck(DeviceID, Nonce, AckSignature)`.
11. **Payment Release**  
    Blockchain → Relay: Transfer funds to Relay’s wallet.

### Phase 3 — Data Retrieval
12. **Data Access**  
    Company → Blockchain: Request encrypted payload.
13. **Decryption**  
    Company: Decrypt using manufacturing-provisioned AES key.

---

## 5. Smart Contract Interface
```solidity
function submitPayload(
    bytes32 deviceId,
    bytes32 nonce,
    bytes32 payloadHash,
    bytes calldata encryptedPayload
) external returns (bytes32 ackChallengeToken);

function confirmAck(
    bytes32 deviceId,
    bytes32 nonce,
    bytes calldata ackSignature
) external returns (bool success);
```

## 6. Data Structures
### Metadata Packet
```JSON
{
  "deviceId": "string",
  "nonce": "hex-string",
  "payloadHash": "hex-string",
  "payloadSize": 12345,
  "signature": "hex-string"
}
```
### Ack Challenge Token
- Issued by the blockchain upon receiving payload.
- Must be signed by the Device before payment is released.

## 7. Payment Model
- Escrowed Payment: Upon submitPayload, payment is locked in escrow.
- Release Trigger: On confirmAck, escrow is released to Relay.
- Failed ACK: If confirmAck is not received within timeout, payment is not released.

## 8. Security Considerations
- All cryptographic operations should use audited libraries.
- ACK signature validation prevents malicious relay spoofing.
- Nonce ensures replayed payloads are rejected.
- Blockchain should store hashes, not full raw data, unless compressed storage is viable.

## 9. Future Enhancements
- Multi-hop offline relaying.
- Incentive adjustments based on payload size or urgency.
- Zero-knowledge proof integration for metadata privacy.
