# MeshPay — UPI Offline Mesh (Demo)

A project where I tried to build a backend for **offline UPI-style payments using a Bluetooth-like mesh network**.

The basic idea was: suppose I'm somewhere with no internet — for example, a basement. I want to send ₹500 to someone nearby. Instead of waiting for the internet to come back, the payment gets encrypted on my phone and passed between nearby phones until one of those phones eventually gets internet access. That phone acts as a bridge and uploads the payment to the backend, where it gets verified and settled.

This project is mainly something I built to understand and practice a few concepts together:

* Encryption and authenticated data
* RSA + AES hybrid encryption
* Mesh/gossip-style communication
* Idempotency
* Concurrent requests
* Replay/tampering protection
* Database transactions
* Optimistic locking
* Spring Boot REST APIs

**This is not a real offline UPI implementation.** I intentionally kept a lot of the real-world complexity out so that I could focus on understanding the backend and security side of the idea.

The mesh itself is also simulated in software, so the entire thing can be demonstrated on one laptop without needing multiple phones or Bluetooth hardware.

---

## What I Actually Built

The project has two main parts:

### 1. The backend

The backend is built using **Spring Boot and Java 17**.

It receives a payment packet from a device that has finally got internet access, decrypts it, checks whether it is valid and recent, prevents duplicate processing, and then updates the sender/receiver balances.

### 2. A simulated mesh network

Instead of using actual Bluetooth devices, I created virtual phones.

A payment can move like this:

```text
Alice's phone
     ↓
 Stranger 1
     ↓
 Stranger 2
     ↓
 Bridge phone
     ↓
 Internet
     ↓
 Spring Boot Backend
     ↓
 Settlement
```

The simulator lets me reproduce this whole flow from the dashboard.

---

## Why I Made This

The main reason for making this project was to understand what actually happens when you combine:

* encryption
* unreliable/offline communication
* duplicate requests
* concurrent processing
* database transactions

The part I found most interesting was **idempotency**.

If three different phones carry the same payment and all of them get internet around the same time, the backend could potentially receive the same payment three times.

Obviously, I don't want:

```text
₹500
₹500
₹500
```

to be deducted from the sender.

I want:

```text
₹500 → settled once
duplicate → ignored
duplicate → ignored
```

So I made that one of the main things this project demonstrates.

---

# How to Run It

## Requirements

You only need:

* JDK 17+
* That's basically it.

There is no separate database or Redis setup required because this version uses H2 and an in-memory idempotency store.

## Windows

Open a terminal in the project folder:

```bash
mvnw.cmd spring-boot:run
```

The first run downloads Maven and the project dependencies, so it can take a couple of minutes.

## Mac/Linux

```bash
./mvnw spring-boot:run
```

## Open the dashboard

Once Spring Boot starts, open:

```text
http://localhost:8080
```

The dashboard is where I run the complete demo.

## Stop the server

```text
Ctrl + C
```

## Run tests

Windows:

```bash
mvnw.cmd test
```

Mac/Linux:

```bash
./mvnw test
```

---

# The Demo

The dashboard is basically designed to let me walk through the complete payment flow.

## 1. Create a payment

I select:

* sender
* receiver
* amount
* PIN

and click **Inject into Mesh**.

The backend creates a `PaymentInstruction`, gives it a unique nonce and timestamp, encrypts it, and puts it inside a `MeshPacket`.

The packet starts with `phone-alice`.

Something like:

```text
PaymentInstruction
        ↓
Hybrid encryption
        ↓
MeshPacket
        ↓
phone-alice
```

The packet also has a TTL so that it doesn't keep travelling through the network forever.

---

## 2. Run the mesh

I can click **Run Gossip Round**.

Every virtual device that has the packet shares it with the other devices in the simulated network.

In this project, "Bluetooth range" basically means all the simulated devices can see each other. Obviously, this is much simpler than what would happen between actual phones.

The TTL gets reduced as the packet moves through the network.

---

## 3. Bridge gets internet

Eventually one of the devices acts as the bridge.

In the default setup, `phone-bridge` has:

```text
hasInternet = true
```

When I click **Bridges Upload to Backend**, that device sends its packets to:

```text
POST /api/bridge/ingest
```

The backend then processes the packet.

The flow is:

```text
Receive packet
      ↓
SHA-256 ciphertext hash
      ↓
Check idempotency
      ↓
Decrypt
      ↓
Check timestamp
      ↓
Database transaction
      ↓
Debit sender
      ↓
Credit receiver
      ↓
Create transaction record
```

The dashboard shows the balance changes and the transaction ledger.

---

# The Part I Spent the Most Time On: Duplicate Payments

This was probably the most important problem I wanted to solve.

Imagine three bridge phones have the same packet:

```text
Bridge 1 ─┐
Bridge 2 ─┼──→ Backend
Bridge 3 ─┘
```

If all three upload the packet at almost exactly the same time, the backend might receive:

```text
Request 1 → ₹500
Request 2 → ₹500
Request 3 → ₹500
```

If I simply processed every request, the sender could lose ₹1500.

That's obviously not acceptable.

### What I did

Before decrypting or settling the payment, I calculate:

```text
SHA-256(ciphertext)
```

Then I try to claim that hash.

The current implementation uses:

```java
Instant prev = seen.putIfAbsent(packetHash, now);
return prev == null;
```

`ConcurrentHashMap.putIfAbsent()` is atomic, which means multiple threads can't all successfully claim the same packet.

So:

```text
Request 1 → claims hash → continues
Request 2 → duplicate → stops
Request 3 → duplicate → stops
```

Only the first request gets to settlement.

I also added a unique index on `packet_hash` in the transaction table as another layer of protection.

---

# Why I Hash the Ciphertext

I initially considered using the packet ID for identifying duplicates, but that isn't a great choice.

The packet ID is part of the outer packet structure and could potentially be changed by an intermediate device.

I also didn't want to decrypt the payment just to figure out whether I'd already seen it.

So the flow is:

```text
ciphertext
    ↓
SHA-256
    ↓
idempotency check
    ↓
decrypt only if it's new
```

This also means the backend can reject obvious duplicates before doing the more expensive RSA/AES decryption work.

---

# Encryption

One of the things I wanted to properly understand while making this was **hybrid encryption**.

I didn't use RSA to encrypt the entire payment because RSA isn't really meant for encrypting arbitrary-sized payloads.

Instead, each packet gets a new AES key.

The process is:

```text
Generate AES-256 key
        ↓
Encrypt payment JSON using AES-GCM
        ↓
Encrypt AES key using RSA-OAEP
        ↓
Put both together
```

The resulting structure is roughly:

```text
[RSA encrypted AES key]
        +
[IV]
        +
[AES-GCM ciphertext + authentication tag]
```

The server has the RSA private key, while devices/intermediaries only deal with the encrypted packet.

So if a random phone forwards the packet, it doesn't get access to:

```text
sender
receiver
amount
PIN hash
```

It just carries the ciphertext.

### Why AES-GCM?

I also wanted tampering to be detectable.

AES-GCM provides authenticated encryption, so if someone changes the ciphertext, the authentication tag won't match and decryption fails.

That gives me:

```text
Original packet → decrypts successfully

Modified packet → authentication failure → rejected
```

---

# Replay Protection

Another problem I looked at was replay attacks.

For example, someone could capture a valid packet and try to send the exact same packet again later.

I used two checks for this.

### Timestamp

The payment contains `signedAt`.

The backend rejects packets older than 24 hours.

Since the timestamp is inside the encrypted/authenticated payload, changing it would break the GCM authentication.

### Nonce

Every payment also gets a unique UUID nonce.

So if Alice actually sends Bob ₹100 twice:

```text
Payment 1 → nonce A
Payment 2 → nonce B
```

They are different payments.

But if someone replays the exact same packet:

```text
Payment 1 → same ciphertext
Payment 1 again → same ciphertext
```

the ciphertext hash is already present in the idempotency store, so it gets rejected as a duplicate.

---

# Project Structure

```text
upi-offline-mesh/
├── pom.xml
├── mvnw
├── mvnw.cmd
├── README.md
│
└── src/
    ├── main/
    │   ├── resources/
    │   │   ├── application.properties
    │   │   └── templates/
    │   │       └── dashboard.html
    │   │
    │   └── java/com/demo/upimesh/
    │       ├── UpiMeshApplication.java
    │       │
    │       ├── model/
    │       │   ├── Account.java
    │       │   ├── AccountRepository.java
    │       │   ├── Transaction.java
    │       │   ├── TransactionRepository.java
    │       │   ├── MeshPacket.java
    │       │   └── PaymentInstruction.java
    │       │
    │       ├── crypto/
    │       │   ├── ServerKeyHolder.java
    │       │   └── HybridCryptoService.java
    │       │
    │       ├── service/
    │       │   ├── DemoService.java
    │       │   ├── VirtualDevice.java
    │       │   ├── MeshSimulatorService.java
    │       │   ├── IdempotencyService.java
    │       │   ├── SettlementService.java
    │       │   └── BridgeIngestionService.java
    │       │
    │       ├── controller/
    │       │   ├── ApiController.java
    │       │   └── DashboardController.java
    │       │
    │       └── config/
    │           └── AppConfig.java
    │
    └── test/
        └── java/com/demo/upimesh/
            └── IdempotencyConcurrencyTest.java
```

---

# What Each Part Does

### `model/`

Contains the main objects used by the application.

`Account` represents an account and balance.

`Transaction` is the settlement ledger.

`MeshPacket` is the packet that moves around the simulated network.

`PaymentInstruction` is what the backend gets after decrypting the packet.

### `crypto/`

This is where I kept the encryption-related code.

`HybridCryptoService` handles:

* AES-256-GCM
* RSA-OAEP
* encryption/decryption
* ciphertext hashing

`ServerKeyHolder` generates the RSA key pair when the application starts.

### `service/`

This contains most of the actual logic.

The important one is:

```text
BridgeIngestionService
```

The whole backend pipeline is basically:

```text
hash
 → claim
 → decrypt
 → freshness check
 → settle
```

`SettlementService` handles the actual balance changes and transaction creation.

`MeshSimulatorService` is responsible for moving packets between the virtual phones.

### `controller/`

Contains the REST endpoints and dashboard controller.

---

# API

| Method | Endpoint             | Purpose                     |
| ------ | -------------------- | --------------------------- |
| GET    | `/`                  | Dashboard                   |
| GET    | `/api/server-key`    | Get server public key       |
| GET    | `/api/accounts`      | Get accounts/balances       |
| GET    | `/api/transactions`  | Get recent transactions     |
| GET    | `/api/mesh/state`    | Get current mesh state      |
| POST   | `/api/demo/send`     | Create and inject a payment |
| POST   | `/api/mesh/gossip`   | Run one gossip round        |
| POST   | `/api/mesh/flush`    | Upload bridge packets       |
| POST   | `/api/mesh/reset`    | Reset the demo              |
| POST   | `/api/bridge/ingest` | Receive a bridge packet     |
| GET    | `/h2-console`        | H2 database console         |

The endpoint I'd consider the actual backend part of the project is:

```text
POST /api/bridge/ingest
```

The other mesh endpoints mainly exist to make the demo possible on one machine.

---

# Tests

I added a few tests around the parts I considered most important.

### `encryptDecryptRoundTrip`

Makes sure data encrypted using the hybrid encryption implementation can be decrypted correctly.

### `tamperedCiphertextIsRejected`

Changes a byte in the ciphertext and checks that the backend rejects it instead of processing it.

### `singlePacketDeliveredByThreeBridgesSettlesExactlyOnce`

This is the main test.

Three threads try to deliver the same packet at the same time.

The expected result is:

```text
1 × SETTLED
2 × DUPLICATE_DROPPED
```

and the sender's balance should only change once.

---

# Things I Know Are Not Production-Ready

This is probably the most important section of the README because this project is **a concept/demo, not a production payment system**.

There are quite a few things I'd have to replace before something like this could actually be used.

| Current project                      | What I'd use in production               |
| ------------------------------------ | ---------------------------------------- |
| H2 in-memory database                | PostgreSQL/MySQL with proper replication |
| `ConcurrentHashMap`                  | Redis `SET NX EX`                        |
| RSA key generated at startup         | HSM / AWS KMS / Vault                    |
| Server creates the payment           | Actual Android/Kotlin client             |
| Simulated mesh                       | Real BLE / Wi-Fi Direct                  |
| Demo settlement service              | Actual bank/NPCI integration             |
| No authentication on bridge endpoint | mTLS / signed bridge certificates        |
| Seeded accounts                      | Real KYC'd accounts and VPAs             |
| H2 console available                 | Completely disabled                      |
| No rate limiting                     | Rate limits + velocity checks            |
| Console logging                      | Structured logs + monitoring/SIEM        |

The biggest difference is that the **core idea can be demonstrated with this project, but the infrastructure around it would need to be completely different**.

---

# What I'd Do If I Took This to Production

If I were continuing this project instead of keeping it as a college/portfolio project, my next steps would roughly be:

### 1. Build the actual mobile client

The current sender is simulated by the backend.

I'd move that part into an Android app, probably using Kotlin.

The phone would actually:

* create the payment
* encrypt it
* store it locally
* discover nearby devices
* forward packets
* detect a bridge device

### 2. Replace the simulated mesh

`MeshSimulatorService` is useful for testing the concept, but it isn't a real Bluetooth implementation.

I'd need to investigate:

* BLE GATT
* background restrictions on Android
* Wi-Fi Direct
* device discovery
* battery consumption
* packet size
* unreliable connections
* retry logic

This is actually one of the areas where I expect the real implementation to be much harder than the simulation.

### 3. Replace the in-memory infrastructure

I'd move:

```text
H2 → PostgreSQL
ConcurrentHashMap → Redis
```

and run the backend across multiple instances.

The idempotency mechanism would then need to work across all backend replicas, not just inside one JVM.

### 4. Secure bridge devices

Right now, the bridge endpoint doesn't have proper authentication.

A production version would need to know:

> "Is this actually an approved bridge device?"

I'd look at mutual TLS or signed device certificates.

### 5. Proper key management

The current application generates the RSA key when it starts.

That's fine for a demo.

It definitely isn't how I'd handle a real payment system.

The private key would need proper protected storage, such as an HSM/KMS/Vault setup, with key rotation and access controls.

### 6. Real payment/ledger integration

The current accounts and balances are just database records created for the demo.

A real implementation would need to integrate with an actual banking/payment system rather than having my Spring Boot service act as the bank ledger.

---

# Limitations I Can't Really Code My Way Out Of

Some of the limitations aren't bugs in my implementation. They're problems with the basic idea of trying to move money without having connectivity.

## 1. The sender's balance can't really be verified offline

This is the biggest issue.

Suppose Alice has ₹500 in her actual bank account.

She sends Bob ₹500 while completely offline.

From Bob's point of view, the payment hasn't actually been settled yet.

Alice could potentially have no money by the time the packet finally reaches the backend.

So this system is closer to:

```text
"I have created a payment instruction that will
eventually be settled."
```

than:

```text
"₹500 has definitely moved right now."
```

A real offline payment system needs some form of **pre-funded / hardware-backed offline value** so that the phone can prove that the money was available before going offline.

---

## 2. Offline double spending is possible

Suppose Alice has ₹500.

While offline:

```text
Alice → Bob      ₹500
Alice → Carol    ₹500
```

She can potentially create both payment packets.

The backend can't know which one was created first until the packets eventually reach it.

So one might settle and the other might fail.

This is a fundamental problem with trying to perform account-based payments without a shared online state.

---

## 3. Real Bluetooth communication is much harder

My simulator basically assumes:

```text
Everyone can communicate with everyone.
```

Real phones obviously don't work like that.

Things I'd have to deal with include:

* BLE background restrictions
* Android/iOS differences
* device discovery
* permissions
* connection failures
* battery usage
* packet size
* devices leaving the network
* unreliable forwarding

So the mesh simulator proves the **logic of the idea**, not that the Bluetooth implementation is solved.

---

## 4. Privacy and metadata

The actual payment contents are encrypted, so an intermediate device can't simply read the amount or receiver.

But that device still knows that it is carrying some payment packet.

In a real deployment I'd have to think about:

* metadata privacy
* device seizure
* data retention
* regulatory requirements
* liability when someone else's device carries a payment

---

# What This Project Is Actually Meant to Be

I don't want to present this as:

> "I built offline UPI."

That would be misleading.

A more accurate description is:

> **A Spring Boot prototype for mesh-routed deferred payment settlement, built to explore encryption, gossip-based forwarding, idempotent processing, replay protection, and transactional settlement.**

The mesh and payment system are simplified because the main purpose of the project was **learning and experimentation**.

The things I was mainly trying to understand were:

```text
Can I securely pass a payment through devices
I don't trust?

        ↓

Can I detect if the packet was modified?

        ↓

What happens if the same payment reaches
the backend multiple times?

        ↓

Can I make concurrent duplicate requests
settle only once?

        ↓

Can I keep the balance update and ledger
entry in one transaction?
```

For me, those were the interesting engineering problems.

There are obviously limitations, especially around actual offline fund verification, double spending, real Bluetooth networking, authentication, key management, and banking integration.

But as a **college/portfolio project**, the goal was to take a complicated real-world problem, simplify the parts I couldn't realistically build yet, and use the simplified version to understand the underlying backend and security concepts.

---

# Troubleshooting

### `java: command not found`

Install JDK 17+ and make sure Java is available in your PATH.

### Port 8080 is already being used

Change `server.port` in `application.properties`.

### `mvnw.cmd` isn't recognized in PowerShell

Use:

```powershell
.\mvnw.cmd spring-boot:run
```

### First Maven run takes a long time

The first run downloads Maven and the dependencies. It should be much faster after that.

### Concurrency test fails occasionally

The concurrency test is timing-sensitive. If it fails once, run it again. If it keeps failing, I'd investigate the actual failure rather than assuming the test is broken.

---

# License

This is demo/learning code with no license.

Feel free to use it for learning and experimentation.

