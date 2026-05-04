# 📡 UPI Without Internet — Offline Mesh Payment System

> **Offline-first UPI payments routed via Bluetooth-style mesh networking, with end-to-end encryption and guaranteed exactly-once settlement.**

[![Java](https://img.shields.io/badge/Java-17+-orange?style=flat-square&logo=java)](https://adoptium.net/)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.3-brightgreen?style=flat-square&logo=springboot)](https://spring.io/projects/spring-boot)
[![Build](https://img.shields.io/badge/Build-Maven%20Wrapper-blue?style=flat-square&logo=apachemaven)](https://maven.apache.org/)
[![License](https://img.shields.io/badge/License-Open%20Demo-lightgrey?style=flat-square)]()

---

## 🧠 What Problem Does This Solve?

India has 500M+ UPI users — but network outages, underground areas, and rural dead zones still break payments. This project demonstrates a **backend system where a UPI payment survives with zero internet**, using phones around you as anonymous packet carriers.

> **Scenario:** You're in a basement with no signal. You initiate ₹500 to a friend. Your phone encrypts it and broadcasts it over Bluetooth. Strangers' phones hop it forward. When *any one* of them gets connectivity, it's uploaded, decrypted, verified, and settled — exactly once.

---

## ✨ Key Technical Highlights

| Challenge | Solution Implemented |
|---|---|
| Untrusted intermediaries can read/tamper data | **Hybrid RSA-OAEP + AES-256-GCM** encryption (TLS-style) |
| Same packet delivered by 3 bridges simultaneously | **Atomic `putIfAbsent`** on SHA-256 ciphertext hash (idempotency) |
| Replay attacks with old captured packets | **Signed timestamp** inside encrypted payload + 24-hour freshness window |
| Double settlement race condition | **Optimistic locking** (`@Version`) on Account entity + DB-level unique index |
| No internet, no real Bluetooth hardware needed | Full **software mesh simulator** with gossip protocol — runs on a single laptop |

---

## 🏗️ Architecture

```
┌──────────────────────────────────────────────────────────┐
│               SENDER PHONE (offline)                     │
│  PaymentInstruction { sender, receiver, amount, nonce }  │
│         │                                                │
│         ▼  Hybrid Encrypt (RSA-OAEP + AES-256-GCM)      │
│  MeshPacket { packetId, ttl, createdAt, ciphertext }     │
└───────────────────────┬──────────────────────────────────┘
                        │  Bluetooth Gossip (TTL-decremented hops)
          ┌─────────────┼─────────────┐
          ▼             ▼             ▼
      [stranger1]   [stranger2]   [bridge]  ← gets 4G, walks outside
                                     │
                                     ▼  HTTPS POST /api/bridge/ingest
┌──────────────────────────────────────────────────────────┐
│              SPRING BOOT BACKEND (this project)          │
│                                                          │
│  1. SHA-256(ciphertext) hash                             │
│  2. Claim hash via atomic putIfAbsent → reject duplicate │
│  3. RSA-OAEP decrypt AES key → AES-GCM decrypt payload  │
│  4. Verify freshness (signedAt < 24h ago)                │
│  5. @Transactional: debit sender + credit receiver       │
└──────────────────────────────────────────────────────────┘
```

---

## 🔐 Cryptography Deep-Dive

### Why Hybrid Encryption?
RSA can only encrypt ~245 bytes with a 2048-bit key — too small for a JSON payload. The standard solution (used by TLS itself) is:

1. Generate a **fresh AES-256 key** per packet
2. Encrypt the payload with **AES-256-GCM** (fast + authenticated)
3. Encrypt just the AES key with **RSA-OAEP** (small, secure)
4. Concatenate: `[256B RSA-wrapped key] + [12B IV] + [AES ciphertext + 16B GCM tag]`

**Why GCM specifically?** It's authenticated encryption. Flipping a single bit in transit causes decryption to throw — the GCM tag doesn't verify. Tampered packets are cryptographically rejected before touching the ledger.

---

## ⚙️ Idempotency — The Core Engineering Problem

Three bridge phones carry the same packet. All three walk outside simultaneously. Without idempotency, the sender gets debited ₹1500.

**The fix:**

```java
// IdempotencyService.java
Instant prev = seen.putIfAbsent(packetHash, now);
return prev == null; // Only ONE thread wins this race
```

`ConcurrentHashMap.putIfAbsent` is atomic — even with 100 threads firing at the same nanosecond, only one returns `null`. The rest are short-circuited as `DUPLICATE_DROPPED` before any decryption or DB work.

**Defense in depth:** `transactions.packet_hash` also has a database-level `UNIQUE` index. If the in-memory cache ever misses (e.g. after a restart), the DB acts as a final guardrail.

> In production, `ConcurrentHashMap` → **Redis `SET NX EX`** for distributed idempotency across replicas.

---

## 🚀 Getting Started

### Prerequisites
- **JDK 17+** (verify with `java -version`)
- No database, no Redis, no separate Maven install required

### Run (Windows)
```bash
mvnw.cmd spring-boot:run
```

### Run (Mac / Linux)
```bash
./mvnw spring-boot:run
```

First run downloads Maven + dependencies (~90 MB). Subsequent starts take ~5 seconds.

### Open Dashboard
```
http://localhost:8080
```

### Run Tests
```bash
mvnw.cmd test
```

---

## 🖥️ Interactive Demo Walkthrough

The dashboard lets you drive the entire payment lifecycle from a single browser tab:

| Step | Button | What Happens |
|---|---|---|
| 1 | **📤 Inject into Mesh** | Sender phone encrypts payment, hands to `phone-alice` |
| 2 | **🔄 Run Gossip Round** | Packet hops across all virtual devices, TTL decrements |
| 3 | **📡 Bridges Upload** | Bridge node gets "4G", POSTs packet to backend |
| 4 | **Verify Idempotency** | Run test: 3 threads deliver same packet → exactly 1 settles |

---

## 📁 Project Structure

```
src/main/java/com/demo/upimesh/
│
├── crypto/
│   ├── ServerKeyHolder.java          # RSA-2048 keypair generation on startup
│   └── HybridCryptoService.java      # RSA-OAEP + AES-256-GCM encrypt/decrypt
│
├── service/
│   ├── BridgeIngestionService.java   # THE pipeline: hash → claim → decrypt → settle
│   ├── IdempotencyService.java       # ConcurrentHashMap-based atomic deduplication
│   ├── SettlementService.java        # @Transactional debit/credit with optimistic lock
│   ├── MeshSimulatorService.java     # Gossip protocol across virtual devices
│   └── VirtualDevice.java            # Simulated phone node in the mesh
│
├── model/
│   ├── Account.java                  # JPA entity with @Version (optimistic locking)
│   ├── Transaction.java              # Ledger with UNIQUE constraint on packetHash
│   ├── MeshPacket.java               # Wire format (outer fields + opaque ciphertext)
│   └── PaymentInstruction.java       # Decrypted payload (sender/receiver/amount/nonce)
│
└── controller/
    ├── ApiController.java            # All REST endpoints
    └── DashboardController.java      # Serves dashboard HTML at /
```

---

## 🔌 API Reference

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/` | Interactive demo dashboard |
| `POST` | `/api/demo/send` | Simulate sender phone — encrypt & inject packet |
| `POST` | `/api/mesh/gossip` | Run one gossip round across the mesh |
| `POST` | `/api/bridge/ingest` | **Production endpoint** — bridge uploads packet |
| `GET` | `/api/accounts` | Current account balances |
| `GET` | `/api/transactions` | Last 20 settled transactions |
| `GET` | `/api/mesh/state` | State of all virtual devices |
| `POST` | `/api/mesh/reset` | Clear mesh + idempotency cache |

### `POST /api/bridge/ingest` — Request & Response

```json
// Request
{
  "packetId": "550e8400-e29b-41d4-a716-446655440000",
  "ttl": 2,
  "createdAt": 1730000000000,
  "ciphertext": "<base64-encoded RSA+AES blob>"
}

// Response
{
  "outcome": "SETTLED",         // or "DUPLICATE_DROPPED" | "INVALID"
  "packetHash": "a3f8c9...",
  "transactionId": 42,
  "reason": null                // populated on INVALID
}
```

---

## 🧪 Tests

```bash
mvnw.cmd test
```

| Test | What It Verifies |
|---|---|
| `encryptDecryptRoundTrip` | Hybrid encryption is correctly symmetric |
| `tamperedCiphertextIsRejected` | A single bit-flip → `INVALID`, never settles |
| `singlePacketDeliveredByThreeBridgesSettlesExactlyOnce` | **3 concurrent threads, 1 packet → exactly 1 `SETTLED`, 2 `DUPLICATE_DROPPED`** |

---

## 🛠️ Tech Stack

| Layer | Technology |
|---|---|
| Language | Java 17 |
| Framework | Spring Boot 3.3 |
| Persistence | Spring Data JPA + H2 (in-memory) |
| Cryptography | Java `javax.crypto` — RSA-OAEP, AES-256-GCM, SHA-256 |
| Concurrency | `ConcurrentHashMap`, `@Transactional`, `@Version` (JPA optimistic lock) |
| Build | Maven Wrapper (no install needed) |
| Frontend | Vanilla HTML/CSS/JS dashboard (Thymeleaf) |

---

## 🔭 Production Upgrade Path

This is a fully-functional demo. Here's what changes at scale:

| Demo | Production Equivalent |
|---|---|
| H2 in-memory DB | PostgreSQL with replicas |
| `ConcurrentHashMap` idempotency | Redis `SET NX EX 86400` |
| RSA keypair generated at startup | Private key in HSM (AWS KMS / HashiCorp Vault) |
| Software mesh simulator | Real BLE GATT / Wi-Fi Direct on Android |
| Server-side payment creation | Same crypto code ported to Kotlin/Android |
| No auth on `/api/bridge/ingest` | Mutual TLS + signed bridge-node certificates |
| Console logging | Structured logs to SIEM, alerts on `INVALID` spikes |

The **cryptography and idempotency logic is production-shaped**. The infrastructure layer is what changes.

---

## ⚠️ Known Limitations (by Design)

Being transparent about what this demo intentionally does not solve:

1. **No offline balance proof** — The receiver gets an IOU, not a settled payment. If the sender's account is empty at settlement time, the payment fails. (Real-world fix: UPI Lite's pre-funded hardware wallet.)
2. **Offline double-spend is possible** — A sender with ₹500 could hand ₹500 packets to two different people in two different basements. First one to settle wins; the second fails.
3. **Bluetooth in production is hard** — Background BLE on Android 8+ is heavily throttled. This demo sidesteps the problem entirely with a software simulator.

---

## 📬 Contact

Built by **Anurag** — open to feedback, discussions, and opportunities!

- GitHub: [@anurag-00008](https://github.com/anurag-00008)
- Project: [UPI-WITHOUT-INTERNET](https://github.com/anurag-00008/UPI-WITHOUT-INTERNET)

---

> *"The cryptography and idempotency code here is essentially production-shaped. The concept is real — this is how you'd design payments for the next billion users who live where connectivity isn't guaranteed."*
