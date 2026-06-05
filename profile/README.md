<h1 align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="echo-logo-text.png" width="300">
    <img src="public/echo-logo-text.png" alt="Echo  Encrypted Messaging" width="300">
  </picture>
</h1>

# Echo Protocol Organization

Echo is an end-to-end encrypted messaging platform composed of three coordinated repositories:

- `Echo-Chat`: the user-facing web, desktop, and mobile client.
- `Echo-Backend`: the opaque relay, API, realtime, device, group, and media server.
- `Echo-Protocol`: the cryptographic protocol, Rust/WebAssembly primitives, protocol orchestration, formal models, and test vectors.

The project implements private messaging in which message contents and private keys stay on user devices. The backend is intentionally unable to decrypt messages: it stores public key material, encrypted envelopes, routing metadata, and short-lived operational state only.

## System Summary

Echo is built around three ideas:

1. The client is the security boundary. It generates keys, runs the protocol, encrypts messages, decrypts messages, and stores private protocol state locally.
2. The backend is an opaque transport. It authenticates users and devices, stores public key bundles, queues encrypted envelopes, and fans out events, but never receives plaintext message content or private keys.
3. The protocol is tested as a first-class product component. Echo combines implementation tests, known-answer vectors, and symbolic formal verification with ProVerif and Tamarin.

## Security Model

Echo assumes a strong network adversary and an honest-but-curious server:

- The network may intercept, reorder, replay, drop, or modify traffic.
- The server may inspect stored ciphertexts, public keys, routing metadata, and timing information.
- Private keys and plaintext messages must never leave the client device.
- Normal operation assumes uncompromised endpoints.
- Out of scope: operating-system compromise, physical device compromise, hardware side channels, and full malware control of the client.

Security goals include:

- Message confidentiality.
- Message integrity.
- Mutual authentication for established sessions.
- Forward secrecy.
- Post-compromise security.
- Replay resistance.
- Group backward secrecy: newly added members cannot read messages sent before joining.
- Group rekeying after membership changes.
- Multidevice isolation: each device is its own cryptographic identity.

## Cryptographic Design

| Layer | Echo design |
| --- | --- |
| Identity and key agreement | X25519 identity keys, signed prekeys, one-time prekeys, and X3DH-style asynchronous session establishment |
| 1:1 messaging | Double Ratchet with per-message keys, DH ratchet steps, skipped-message handling, and AEAD-authenticated headers |
| Group messaging | MLS-inspired TreeKEM group state with epochs, commits, welcome messages, sender secrets, and rekeying on membership changes |
| Symmetric crypto | AES-256-GCM with 96-bit nonces and authenticated additional data |
| Key derivation | HKDF-SHA256 with protocol-specific info labels |
| Signatures | Ed25519/XEdDSA-style signing and verification for authenticated key material |
| Local storage | Encrypted IndexedDB state protected by a DEK derived from the user password with Argon2id |
| Media attachments | Inline encrypted images for small content; encrypted blob containers for video and voice notes |

## Core Flows

### Registration And Device Provisioning

1. `Echo-Chat` initializes the `Echo-Protocol` WASM module.
2. The device generates local identity material and prekeys.
3. Only public key material is published to `Echo-Backend`.
4. The backend stores key bundles and device metadata in MongoDB.
5. Private keys remain in the client local encrypted database.

### 1:1 Session Establishment

1. Alice requests Bob's public prekey bundle from `Echo-Backend`.
2. Alice runs X3DH locally using `Echo-Protocol`.
3. Alice derives the initial root key and initializes Double Ratchet state.
4. The first encrypted message carries the header material Bob needs to derive the same session.
5. Bob completes the session locally when receiving the first envelope.

The backend only sees a public-key lookup and an opaque encrypted envelope.

### 1:1 Message Delivery

1. The sender serializes the message payload.
2. `Echo-Protocol` derives a fresh message key through the Double Ratchet.
3. The payload is encrypted with AES-256-GCM.
4. `Echo-Chat` sends an opaque envelope to `Echo-Backend`.
5. `Echo-Backend` stores the envelope in the target device mailbox and emits it over Socket.IO if the target device is connected.
6. The receiver decrypts locally and advances its ratchet state.

### Group Messaging

1. A group creator initializes a TreeKEM-style group state.
2. Initial members receive welcome material encrypted for their devices.
3. Each group state has an epoch.
4. Add, remove, and update operations produce commits that advance the epoch.
5. Group messages are encrypted under application secrets derived for the current epoch and sender.

The server manages group metadata and encrypted transport, but not group secrets.

### Multidevice Sync

Echo treats every device as an independent cryptographic identity instead of sharing one master private key across devices. Device pairing uses an explicit confirmation step, then transfers only the state required by the protocol through encrypted device-sync envelopes. Devices can be registered and revoked independently.

### Media Attachments

Echo uses two attachment strategies:

- Images: compressed client-side and embedded inline in the encrypted message payload.
- Video and voice notes: encrypted once with a fresh 256-bit blob key, uploaded as opaque ciphertext, and referenced by a small encrypted descriptor inside the E2E message.

For blob media, the backend stores only ciphertext and an unguessable media identifier. The blob key is delivered only inside the end-to-end encrypted message payload.

## `Echo-Protocol`

`Echo-Protocol` is the cryptographic and verification repository. It provides the protocol implementation consumed by the client and the evidence that the design behaves as specified.

### Implementation Layers

1. Rust/WASM core:
   - X25519 Diffie-Hellman.
   - XEdDSA/Ed25519 signing and verification.
   - HKDF-SHA256 extract/expand.
   - AES-256-GCM authenticated encryption.
   - Key generation and conversion helpers.

2. JavaScript orchestration:
   - X3DH session setup.
   - Double Ratchet state machines.
   - One-time prekey handling.
   - MLS-TreeKEM group state.
   - Welcome messages and commits.
   - Multidevice sync envelopes.
   - Encrypted local database integration.

3. Verification and vectors:
   - RFC/NIST known-answer tests for primitives.
   - Echo-specific vectors for ratchets, groups, and protocol key schedules.
   - ProVerif symbolic models.
   - Tamarin explicit-state models.

### Package Boundary

The WASM package is distributed to the client as `@mascaro101/echo-protocol`. The client should call into this package for cryptographic primitives instead of reimplementing crypto in React code.

## Formal Verification

Echo verifies the protocol design separately from the application code.

### ProVerif

The ProVerif suite contains nine symbolic models:

| Model | Area | Main properties |
| --- | --- | --- |
| `echo_x3dh.pv` | X3DH | Confidentiality, forward secrecy, mutual authentication |
| `echo_ratchet_pcs.pv` | Double Ratchet root step | Post-compromise security |
| `echo_x3dh_kci.pv` | X3DH | Key-compromise impersonation resistance |
| `echo_sas_pairing.pv` | Multidevice pairing | History secrecy under MITM, SAS confirmation |
| `echo_welcome.pv` | Group welcome | Epoch secret confidentiality, welcome authenticity |
| `echo_sender_data.pv` | Group sender data | Sender privacy from server/non-members |
| `echo_group_pcs.pv` | TreeKEM removal | Group PCS after member removal |
| `echo_chain_fs.pv` | Symmetric chain | Forward secrecy for bounded chain steps |
| `echo_token_auth.pv` | Backend token model | Token unforgeability |

The suite reports 14 security properties verified, with falsifiability controls that intentionally remove defenses and produce attacks. This guards against vacuous proofs.

### Tamarin

Tamarin complements ProVerif with explicit-state models:

| Model | Area | Purpose |
| --- | --- | --- |
| `echo_x3dh.spthy` | X3DH | Key secrecy and injective agreement across unlimited sessions |
| `echo_sas_pairing.spthy` | Pairing | History secrecy and agreement on pairing key |
| `echo_group_keyschedule.spthy` | MLS/TreeKEM | Commit secret secrecy and group key schedule behavior |
| `echo_double_ratchet.spthy` | Double Ratchet | Explicit-state reasoning about chain evolution |

The formal models verify the design, not the whole deployed application. Implementation correctness is covered separately with test vectors, unit tests, integration tests, and end-to-end tests.

## Testing Strategy

| Area | Tooling | Coverage intent |
| --- | --- | --- |
| Protocol primitives | Rust tests, Node scripts, known-answer vectors | Byte-for-byte agreement with RFC/NIST vectors |
| Protocol composition | Vitest and Echo vectors | X3DH, Double Ratchet, group crypto, AAD, key schedule, fail-closed behavior |
| Formal design | ProVerif, Tamarin | Symbolic secrecy, authentication, FS, PCS, KCI, pairing, group properties |
| Backend unit/contract | Node `node:test`, supertest | Services, policies, OpenAPI, error shape, environment validation |
| Backend HTTP E2E | supertest + MongoDB test instance | Auth, users, contacts, messages, groups, calls, keys, sync, media, admin/public APIs |
| Backend sockets | Node tests | Socket auth guards, fanout, ACK contracts, message counters, MLS group flows |
| Frontend unit/integration | Vitest, jsdom, fake-indexeddb | Forms, hooks, socket lifecycle, UI behavior |
| Frontend E2E | Playwright | Register, login, protected routes, dashboard, critical user flows, accessibility |
| Deployment smoke | `scripts/smoke.sh` | Health, register, login, refresh, bundle publish, envelope delivery |

Known reported results:

- Frontend: 36 test files passed.
- Backend: 30 test files across unit, contract, HTTP, socket, and behavior tests.
- Backend deterministic logic coverage examples: `opkPolicy.js` at 100%, `shared/constants.js` at 95.9%, `shared/errors.js` at 81.8%.
- Pre-production benchmark on Render/MongoDB Atlas: `/health` p95 around 73 ms, `/status/services` p95 around 73 ms, login around 596 ms median under bcrypt cost and active rate limiting.