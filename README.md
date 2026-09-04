# Networkless UPI

> A proof-of-concept system that explores how payment instructions could travel through an offline peer-to-peer mesh network and settle once connectivity becomes available.

## Overview

Imagine being in a location with no internet connectivity—a basement, remote area, disaster zone, or crowded event venue.

How can a digital payment still be initiated?

**Networkless UPI** explores this idea through a simulated offline mesh network. A payment instruction is encrypted and propagated across nearby devices using a gossip-style protocol. When any participating device regains internet connectivity, it acts as a bridge and forwards the encrypted transaction to the backend for validation and settlement.

The project focuses on the backend engineering challenges involved in such a system:

* Secure transmission through untrusted intermediate devices
* Duplicate transaction prevention
* Replay attack protection
* Transactional settlement
* Concurrent packet delivery

> **Note:** This is an experimental proof-of-concept and not an official UPI or NPCI implementation.

---

## Problem Statement

Traditional digital payments depend heavily on network connectivity.

But what happens when:

* Mobile internet is unavailable?
* A disaster disrupts communication infrastructure?
* Users are in underground locations?
* Network congestion prevents transactions?
* Devices temporarily lose connectivity?

This project explores a **store-and-forward payment architecture** where transaction packets can travel through nearby devices until one device eventually reaches the internet.

---

## How It Works

```text
┌──────────────────┐
│   Sender Device  │
│                  │
│ Creates Payment  │
└────────┬─────────┘
         │
         │ Encrypt Payment
         ▼
┌──────────────────┐
│   Mesh Network   │
│                  │
│ Device → Device  │
│ Device → Device  │
│ Device → Device  │
└────────┬─────────┘
         │
         │ Internet Available
         ▼
┌──────────────────┐
│   Bridge Device  │
└────────┬─────────┘
         │
         │ HTTPS Request
         ▼
┌─────────────────────────┐
│   Spring Boot Backend   │
│                         │
│  1. Hash Packet         │
│  2. Check Duplicate     │
│  3. Decrypt Payload     │
│  4. Validate Timestamp  │
│  5. Settle Transaction  │
└─────────────────────────┘
```

### Example Flow

1. Alice creates a payment while offline.
2. The payment instruction is encrypted.
3. The encrypted packet is injected into the simulated mesh.
4. Nearby devices propagate the packet using gossip rounds.
5. Multiple devices may hold copies of the same packet.
6. A bridge device regains internet connectivity.
7. The bridge uploads the packet to the backend.
8. The backend validates and processes the transaction.
9. Duplicate copies are rejected using idempotency checks.

---

# Key Engineering Challenges

## 1. Secure Transmission Through Untrusted Devices

Intermediate devices should be able to forward a payment packet without being able to read or modify its contents.

The project uses **Hybrid Encryption**:

* RSA-OAEP for encrypting the AES key
* AES-256-GCM for encrypting the payment payload

```text
Payment Instruction
        │
        ▼
Generate AES Key
        │
        ▼
AES-256-GCM Encrypt Payload
        │
        ▼
RSA-OAEP Encrypt AES Key
        │
        ▼
Encrypted Mesh Packet
```

AES-GCM also provides authenticated encryption, meaning tampered ciphertext can be detected during decryption.

---

## 2. Preventing Duplicate Transactions

In a mesh network, the same packet can reach multiple bridge devices.

For example:

```text
           ┌── Bridge A ──► Backend
Payment ───┤
           ├── Bridge B ──► Backend
           │
           └── Bridge C ──► Backend
```

Without protection, the same payment could potentially be processed multiple times.

To prevent this, the backend:

1. Generates a SHA-256 hash of the ciphertext.
2. Atomically claims the hash.
3. Allows only the first request to continue.
4. Drops subsequent copies.

The implementation uses:

```java
ConcurrentHashMap.putIfAbsent()
```

This provides atomic duplicate detection in the demo environment.

In a distributed production architecture, this could be replaced with Redis:

```text
SET transaction_hash value NX EX 86400
```

Additionally, the transaction ledger contains a unique constraint as a defense-in-depth mechanism.

---

## 3. Replay Attack Protection

An attacker could attempt to resend an old captured payment packet.

To mitigate this, each payment contains:

* A unique nonce
* A timestamp

The backend validates whether the transaction timestamp falls within an acceptable freshness window.

Because these values exist inside the encrypted payload, they cannot be modified without invalidating the AES-GCM authentication tag.

---

# Architecture

```text
                    OFFLINE ENVIRONMENT

┌─────────────┐
│   Sender    │
│   Device    │
└──────┬──────┘
       │
       │ Encrypted Payment Packet
       ▼
┌──────────────────────────────┐
│      Simulated Mesh          │
│                              │
│  Device A → Device B         │
│      ↓          ↓            │
│  Device C → Bridge Device    │
└───────────────┬──────────────┘
                │
                │ Connectivity Restored
                ▼
        ┌───────────────┐
        │ Bridge Device │
        └───────┬───────┘
                │
                ▼
        ┌──────────────────┐
        │ Spring Boot API  │
        └────────┬─────────┘
                 │
        ┌────────▼─────────┐
        │ Idempotency Check│
        └────────┬─────────┘
                 │
        ┌────────▼─────────┐
        │ Decryption Layer │
        └────────┬─────────┘
                 │
        ┌────────▼─────────┐
        │ Validation Layer │
        └────────┬─────────┘
                 │
        ┌────────▼─────────┐
        │ Transaction      │
        │ Settlement       │
        └──────────────────┘
```

---

# Tech Stack

### Backend

* Java 17
* Spring Boot
* Spring Data JPA
* Maven

### Database

* H2 Database

### Security

* RSA-OAEP
* AES-256-GCM
* SHA-256

### Core Concepts

* Hybrid Encryption
* Idempotency
* Optimistic Locking
* Concurrent Processing
* Gossip Protocol
* Mesh Networking
* Transaction Management

---

# Project Structure

```text
Networkless_UPI/
│
├── src/main/
│   │
│   ├── java/
│   │   └── com/demo/upimesh/
│   │
│   │       ├── crypto/
│   │       │   ├── HybridCryptoService.java
│   │       │   └── ServerKeyHolder.java
│   │       │
│   │       ├── service/
│   │       │   ├── MeshSimulatorService.java
│   │       │   ├── BridgeIngestionService.java
│   │       │   ├── IdempotencyService.java
│   │       │   └── SettlementService.java
│   │       │
│   │       ├── controller/
│   │       │
│   │       └── model/
│   │
│   └── resources/
│       ├── application.properties
│       └── templates/
│
├── src/test/
│
├── pom.xml
└── README.md
```

---

# Getting Started

## Prerequisites

Make sure you have:

* Java 17+
* Git

Verify Java installation:

```bash
java -version
```

---

## Clone the Repository

```bash
git clone https://github.com/Utkarsh629/Networkless_UPI.git
```

```bash
cd Networkless_UPI
```

---

## Run the Application

### Mac / Linux

```bash
./mvnw spring-boot:run
```

### Windows

```bash
.\mvnw.cmd spring-boot:run
```

Once the application starts, open:

```text
http://localhost:8080
```

---

# Running Tests

Run the test suite using:

### Mac / Linux

```bash
./mvnw test
```

### Windows

```bash
.\mvnw.cmd test
```

The project includes tests for:

* Encryption and decryption
* Tampered packet rejection
* Concurrent duplicate packet delivery
* Exactly-once transaction settlement

---

# API Endpoints

| Method | Endpoint             | Description                                |
| ------ | -------------------- | ------------------------------------------ |
| GET    | `/`                  | Interactive dashboard                      |
| GET    | `/api/accounts`      | Retrieve account balances                  |
| GET    | `/api/transactions`  | Retrieve transaction history               |
| GET    | `/api/mesh/state`    | View mesh device state                     |
| POST   | `/api/demo/send`     | Create and inject payment                  |
| POST   | `/api/mesh/gossip`   | Run mesh gossip round                      |
| POST   | `/api/mesh/flush`    | Upload packets through bridge              |
| POST   | `/api/mesh/reset`    | Reset mesh state                           |
| POST   | `/api/bridge/ingest` | Production-style bridge ingestion endpoint |

---

# Interesting Test Case: Concurrent Delivery

One of the most important scenarios tested is concurrent delivery.

The same encrypted payment packet is delivered simultaneously by multiple bridge nodes.

```text
Bridge 1 ─────┐
Bridge 2 ─────┼────► Backend
Bridge 3 ─────┘
```

Expected result:

```text
1 Transaction → SETTLED
2 Transactions → DUPLICATE_DROPPED
```

This ensures that a payment is processed **exactly once**, even when multiple copies arrive concurrently.

---

# Production Considerations

This project is a proof-of-concept and intentionally simplifies several production concerns.

| Current Implementation | Production Alternative                |
| ---------------------- | ------------------------------------- |
| H2 Database            | PostgreSQL / MySQL                    |
| ConcurrentHashMap      | Redis                                 |
| Generated RSA Keys     | HSM / Cloud KMS                       |
| Simulated Mesh         | Bluetooth LE / Wi-Fi Direct           |
| In-memory devices      | Real mobile devices                   |
| No authentication      | Mutual TLS / Device Certificates      |
| Local settlement       | Banking / payment network integration |

---

# Limitations

This project demonstrates **mesh-routed deferred settlement**, not a fully decentralized offline payment system.

Some important limitations include:

* The receiver cannot immediately verify whether the sender has sufficient funds.
* A malicious sender could attempt offline double-spending.
* Actual Bluetooth mesh networking introduces platform and battery constraints.
* Real-world payment systems require regulatory and security compliance.
* Settlement only occurs once connectivity is restored.

A production-grade offline payment system would likely require a pre-funded secure wallet or hardware-backed offline balance.

---

# Future Improvements

* [ ] Android client implementation
* [ ] Bluetooth LE communication
* [ ] Wi-Fi Direct fallback
* [ ] Redis-based distributed idempotency
* [ ] PostgreSQL persistence
* [ ] Device authentication
* [ ] Digital signatures
* [ ] Message queue integration
* [ ] Docker support
* [ ] Distributed deployment
* [ ] Real-time mesh visualization

---

# Learning Outcomes

Through this project, I explored:

* Designing secure distributed systems
* Hybrid cryptography
* Idempotency in payment systems
* Concurrent request handling
* Transaction management
* Optimistic locking
* Store-and-forward architectures
* Gossip-based message propagation
* Replay attack mitigation

---

## Disclaimer

This project is built for educational and experimental purposes.

It is **not affiliated with NPCI or the official UPI ecosystem** and should not be used to process real financial transactions.

---

## Author

**Utkarsh Raj**


