# UPI Offline Mesh — NexPay Demo

A Spring Boot backend that demonstrates **offline UPI payments routed through a peer-to-peer, Bluetooth-style mesh network**.

---
## 1. Problem Statement

In modern digital payment ecosystems like UPI, an instantaneous bidirectional online handshake between sender, receiver, NPCI, and the core banking system (CBS) is mandatory. When connectivity fails—such as inside underground basements, remote terrains, or congested stadium environments—payment processing halts completely.

Traditional offline solutions like pre-funded wallets (e.g., UPI Lite) solve this partially, but they require pre-allocation of funds into local secure hardware storage with strict transaction ceilings. They do not enable direct account-to-account settlement routed asynchronously from zero-connectivity zones.

---

## 2. Approach: Store-and-Forward Mesh Routing

NexPay decouples **transmission** from **settlement** using an opportunistic store-and-forward gossip protocol:

1. **Client-Side Hybrid Encryption:** The sender constructs an instruction payload and cryptographically locks it with the central bank's public key. Intermediary nodes carry opaque ciphertext and cannot read or alter amounts, accounts, or recipients.
2. **Proximity-Based Gossip Broadcast:** The sender's phone broadcasts the encrypted packet over local Bluetooth Low Energy (BLE) / Wi-Fi Direct to neighboring peers.
3. **Multi-Hop Data Mule Relay:** Intermediate stranger devices store and forward the packet across physical hops with decremental Time-To-Live (`TTL`) thresholds.
4. **Opportunistic Ingestion:** When any intermediate node (a "bridge node") enters an area with cellular/Wi-Fi connectivity, it silently uploads its buffered packets to the central backend via HTTPS.
5. **Deduplicated Settlement:** The backend deduplicates competing deliveries of the same transaction at wire-speed, decrypts authenticated payloads, validates business rules, and commits atomic ledger updates.

---

## 3. System Architecture & Ingestion Pipeline

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        SENDER PHONE (Offline)                           │
│  PaymentInstruction { sender, receiver, amount, pinHash, nonce, time }  │
│              │                                                          │
│              ▼ Encrypt via Hybrid RSA-OAEP + AES-256-GCM               │
│  MeshPacket { packetId, ttl, createdAt, ciphertext }                    │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │ Bluetooth / Wi-Fi Gossip
                                     ▼
         ┌─────────┐   hop   ┌─────────┐   hop   ┌─────────┐
         │ Node A  │ ──────► │ Node B  │ ──────► │ Bridge  │ (Walks outside, gets 4G)
         └─────────┘         └─────────┘         └────┬────┘
                                                      │
                                                      ▼ HTTPS POST /api/bridge/ingest
┌─────────────────────────────────────────────────────────────────────────┐
│                      SPRING BOOT BACKEND SERVER                         │
│                                                                         │
│  [Step 1: Hash Ciphertext]                                              │
│     Compute SHA-256(ciphertext) to generate an immutable claim key.     │
│                                                                         │
│  [Step 2: Idempotency Claim]                                            │
│     Atomic putIfAbsent() check (JVM ConcurrentHashMap / Redis SETNX).   │
│     Duplicate deliveries drop immediately (0 CPU spent on crypto).      │
│                                                                         │
│  [Step 3: Cryptographic Unwrap & Authentication]                        │
│     RSA-OAEP private key decrypts the ephemeral AES-256 key.            │
│     AES-256-GCM decrypts payload and verifies authentication tag.       │
│     Bit-level tampering triggers instant rejection.                     │
│                                                                         │
│  [Step 4: Freshness Validation]                                         │
│     Verify (now - signedAt) <= 24 Hours. Reject expired packets.        │
│                                                                         │
│  [Step 5: Transactional Settlement]                                     │
│     @Transactional debit/credit execution with optimistic locking.      │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Key Concepts and Security Logic

### Hybrid Cryptography (Confidentiality & Integrity)
- **Why Hybrid?** Asymmetric ciphers like RSA-2048 cannot natively encrypt payloads larger than ~245 bytes.
- **Protocol:**
  1. A fresh, cryptographically secure 256-bit AES key and 12-byte IV are generated per packet.
  2. The payload is encrypted with **AES-256-GCM** (Galois/Counter Mode). GCM provides authenticated encryption; any bit flip in transit invalidates the 16-byte authentication tag, throwing a `BadPaddingException` / `AEADBadTagException`.
  3. The ephemeral AES key is wrapped with the server's **RSA-2048 (OAEP padding)** public key.
  4. Final payload structure: `[256 bytes RSA key][12 bytes IV][AES-GCM ciphertext + 16 bytes tag]`.

### Wire-Speed Idempotency (The Duplicate-Storm Problem)
In a mesh topology, multiple bridge phones may collect the identical packet and flush it to the backend simultaneously upon gaining 4G coverage.
- **The Solution:** Deduplication occurs against `SHA-256(ciphertext)`.
- **Pre-Decryption Filtering:** Hashing the raw ciphertext allows atomic deduplication (`ConcurrentHashMap.putIfAbsent` or Redis `SETNX`) *before* spending CPU cycles on expensive RSA decryption.
- **Tamper-Resistance:** Malicious nodes modifying outer metadata (like `packetId`) cannot bypass deduplication because the ciphertext itself remains identical for identical transactions.
- **Database Defense-in-Depth:** A unique constraint on `transactions.packet_hash` acts as a fail-safe against concurrent settlement race conditions.

### Replay & Tampering Defenses
- **Freshness Window:** Every instruction contains a millisecond-precision `signedAt` timestamp enforced to a 24-hour expiration window.
- **Cryptographic Nonce:** Every transaction includes a UUID nonce ensuring that two successive payments of identical amounts generate entirely distinct ciphertexts.

---

## 5. Repository Structure

```
upi-offline-mesh/
├── pom.xml                                  # Maven dependencies (Spring Boot 3.3, Java 17)
├── mvnw, mvnw.cmd                           # Portable Maven wrappers
└── src/
    ├── main/
    │   ├── resources/
    │   │   ├── application.properties       # H2 DB config, server port, timeouts
    │   │   └── templates/dashboard.html     # Real-time simulation control panel
    │   └── java/com/demo/upimesh/
    │       ├── UpiMeshApplication.java      # Application entry point
    │       ├── model/                       # Data entities: Account, Transaction, MeshPacket
    │       ├── crypto/                      # RSA/AES-GCM Hybrid encryption service
    │       ├── service/                     # Ingestion, Settlement, Idempotency, Simulation
    │       ├── controller/                  # REST APIs and Dashboard endpoints
    │       └── config/                      # Scheduler and thread pool configuration
    └── test/java/com/demo/upimesh/
        └── IdempotencyConcurrencyTest.java  # Multi-threaded concurrent delivery test
```

---

## 6. Getting Started

### Prerequisites
- **Java Development Kit (JDK) 17+** installed and set in your environment (`java -version`).
- No separate database or broker installation required (runs an embedded H2 engine).

### Building and Running

**On Windows:**
```cmd
mvnw.cmd spring-boot:run
```

**On Linux / macOS:**
```bash
chmod +x mvnw
./mvnw spring-boot:run
```

### Accessing the Demo
Once the service prints `Started UpiMeshApplication`, navigate to:
- **Interactive UI Dashboard:** `http://localhost:8080`
- **H2 In-Memory Database Console:** `http://localhost:8080/h2-console` (JDBC URL: `jdbc:h2:mem:upimesh`, Username: `sa`, Password: *(blank)*)

---

## 7. Interactive Demo Workflow

1. **Step 1: Inject Payment** — Generate a signed, encrypted transaction on a simulated offline sender node (`phone-alice`).
2. **Step 2: Gossip Rounds** — Step through peer-to-peer broadcasts across neighboring nodes; track hops and TTL decrement.
3. **Step 3: Ingestion via Bridges** — Trigger bridge devices returning to cellular range to flush buffered packets to the server. Observe atomic account balance updates and the settlement ledger.
4. **Step 4: Concurrency Verification** — Trigger simultaneous flushes from multiple bridge nodes to watch duplicate packets get dropped without double-debiting.

---

## 8. API Reference

| HTTP Method | Endpoint | Description |
|---|---|---|
| `GET` | `/` | Serves the real-time simulation dashboard |
| `GET` | `/api/accounts` | Lists virtual accounts and current balances |
| `GET` | `/api/transactions` | Lists settled transaction history |
| `GET` | `/api/mesh/state` | Returns the state and buffered packets of all simulated devices |
| `POST` | `/api/demo/send` | Simulates an offline sender creating and encrypting a payment |
| `POST` | `/api/mesh/gossip` | Triggers a peer-to-peer broadcast round across simulated nodes |
| `POST` | `/api/mesh/flush` | Simulates bridge nodes uploading packets via cellular uplink |
| `POST` | `/api/bridge/ingest` | **Production ingestion endpoint** for bridge nodes pushing packets |

#### Sample Ingestion Payload (`POST /api/bridge/ingest`)
```json
{
  "packetId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "ttl": 3,
  "createdAt": 1730000000000,
  "ciphertext": "k3f8A...[Base64 Encrypted Blob]..."
}
```

#### Ingestion Response
```json
{
  "outcome": "SETTLED",
  "packetHash": "8d969eef6ecad3c29a3a629280e686cf0c3f5d5a86aff3ca12020c923adc6c92",
  "transactionId": 101,
  "reason": null
}
```

---

## 9. Test Suite

Execute unit and concurrency tests:
```bash
mvnw.cmd test
```

- **`singlePacketDeliveredByThreeBridgesSettlesExactlyOnce`**: Dispatches simultaneous threads delivering the same packet payload to `/api/bridge/ingest`. Asserts exactly 1 settlement and 2 drops.
- **`tamperedCiphertextIsRejected`**: Verifies that bit flips in transit fail AES-GCM verification and return `INVALID`.
- **`encryptDecryptRoundTrip`**: Validates RSA-OAEP + AES-GCM integrity.

---

## 10. Practical Realities & Limitations

| Dimension | Concept Demo | Production Reality |
|---|---|---|
| **Fund Verification** | Deferred settlement (IOU risk if sender has insufficient funds upon upload) | Requires pre-funded hardware-backed secure elements (e.g., UPI Lite) to guarantee solvency offline |
| **Double-Spending** | First packet to reach backend wins; conflicting packets rejected | Solved either by hardware secure enclave counters or offline risk underwriting |
| **P2P Transport** | Simulated in-memory gossip graph | Background BLE connection limits, OS background execution limits (Android Doze, iOS background constraints) |
| **Key Management** | In-memory keypair regenerated on startup | Hardware Security Modules (HSMs), AWS KMS, PKI infrastructure with device key rotation |
| **Idempotency Store** | In-memory `ConcurrentHashMap` | Distributed Redis cluster with multi-region persistence (`SET key NX EX`) |
