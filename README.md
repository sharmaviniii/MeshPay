# MeshPay — Offline-First Distributed Payment Routing System

> **A production-inspired distributed systems project that demonstrates how encrypted digital payments can be securely routed without internet connectivity using a Bluetooth-style mesh network.**

---

## Why does this project exist?

Digital payments stop working when internet connectivity disappears. This project explores how encrypted payment packets can securely hop across nearby devices until an internet-enabled bridge node uploads them for settlement.

Instead of:

Phone → Internet → Backend

MeshPay demonstrates:

Phone → Relay Devices → Bridge Node → Backend

---

## Project's technical relevance

Unlike a traditional CRUD payment application, MeshPay combines:

- Distributed Systems
- Backend Engineering
- Applied Cryptography
- Network Routing
- Concurrency
- Secure Transaction Processing

The project models the complete lifecycle of an offline payment—from encrypted packet creation to exactly-once settlement.

---

## Core Engineering Challenges

### Secure routing through untrusted devices
- RSA-OAEP + AES-256-GCM hybrid encryption
- End-to-end encrypted payloads
- Authenticated encryption prevents tampering

### Exactly-once settlement
- SHA-256 packet hashing
- Atomic idempotency
- Thread-safe concurrent processing

### Replay protection
- Timestamp validation
- Nonce verification
- Freshness checks

---

## Architecture

Sender → Hybrid Encryption → Mesh Packet → Mesh Routing → Bridge Node → Backend → Idempotency → Decryption → Settlement → Ledger

---

## Tech Stack

Java 17 • Spring Boot • Spring Data JPA • REST APIs • H2 • Maven • RSA-OAEP • AES-256-GCM • SHA-256 • JUnit

---

## Current Features

- End-to-end encrypted transactions
- Gossip-based mesh routing simulator
- Bridge-node synchronization
- Exactly-once settlement
- Replay attack protection
- Interactive dashboard
- Concurrency testing

---

## Planned Roadmap

### Near Term
- Real Android BLE communication
- Digital signatures
- PostgreSQL
- Redis-backed idempotency
- Docker support

### Long Term
- Kafka event pipeline
- Mutual TLS
- Prometheus & Grafana
- OpenTelemetry
- Kubernetes deployment
- Multi-region settlement simulation
- Performance benchmarking

---

## Running

```bash
./mvnw spring-boot:run
```

Windows:

```cmd
mvnw.cmd spring-boot:run
```

Tests:

```bash
./mvnw test
```

---

## Vision

MeshPay is a research-oriented distributed systems project demonstrating how resilient payment infrastructure can function in low-connectivity environments through secure multi-hop routing and deferred settlement.
