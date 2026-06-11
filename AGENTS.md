# AGENTS_PAPER.md — Project Brief for IEEE Conference Paper

> **Purpose:** Complete technical reference about this project. An agent reads this file and uses the content to write an IEEE conference paper. No LaTeX instructions here — only project facts.

---

## 1. Paper Identity

| Field | Value |
|-------|-------|
| **Title** | Adaptive Secure Communication System for Unreliable and Adversarial Networks |

---

## 2. What This Project Is

A **multi-client end-to-end encrypted (E2EE) chat system** written in C (C11, Linux/POSIX). It combines:

- Concurrent TCP server using POSIX threads
- TLS 1.3 transport encryption via OpenSSL 3.x
- Application-layer E2EE using the Signal Double Ratchet Algorithm
- Dual-transport delivery over TCP and UDP simultaneously
- A three-state Adaptive Engine that reconfigures crypto and transport based on live network and security metrics
- Persistent offline message queue for asynchronous delivery
- Per-IP Intrusion Detection System (IDS) coupled to the adaptive engine
- GTK3 GUI client and a CLI client

The server listens on TCP port **8080** and UDP port **8081**.

---

## 3. Problem Being Solved

Secure messaging over unreliable and adversarial networks fails when security posture is static. Standard TLS protects the channel but cannot react to:
- Rising packet loss degrading delivery
- Detected replay attacks or authentication brute-force
- Traffic analysis (ciphertext length reveals message length)

This system addresses the gap: **security behavior adapts automatically** based on measured network quality and detected threats. The three-state engine escalates from Normal → Unstable → High-Risk and tunes retries, padding, key rotation frequency, and timing delays accordingly.

---

## 4. Key Contributions

1. **Three-state Adaptive Engine** — Normal / Unstable / High-Risk — driven by live packet loss, RTT, auth failure count, and replay detection count
2. **Signal Double Ratchet** at the application layer, providing per-message forward secrecy and break-in recovery, independent of TLS
3. **Dual-path (TCP+UDP) multipath delivery** with 128-bit message-ID deduplication ring buffer
4. **Lightweight per-IP IDS** that feeds security events directly into the adaptive state machine
5. **Offline message queue** with chronological replay on reconnect
6. **GTK3 GUI client** with directed messaging, priority levels (Normal/Urgent/Critical), and live online-users panel

---

## 5. System Architecture

Two-tier: server process + client processes. All connected over TCP (primary) and UDP (backup).

**Server:**
- Single POSIX process
- On `accept()`, spawns a detached `pthread` per client (`handle_client`)
- Shared client table (`ClientEntry[50]`) protected by `pthread_mutex_t`
- Adaptive Engine runs as a background thread (1-second evaluation tick)
- UDP receiver thread handles backup message copies and presence
- Offline queue stored on disk under `data/offline_queue/<username>/`

**Client:**
- Two threads: send thread (drains priority queue, encrypts, sends) and recv thread (receives, deduplicates, decrypts, fires UI callbacks)
- GTK client: all widget mutations via `gdk_threads_add_idle()` — recv thread never touches GTK directly
- CLI client: `@recipient message` syntax, `@all` for broadcast, `!urgent` / `!critical` prefixes

**Module map:**

| Module | Source | Role |
|--------|--------|------|
| Server | `src/server/server.c`, `client_handler.c`, `room_manager.c`, `auth_manager.c` | Accept loop, routing, auth, user-list broadcast |
| GTK Client | `src/client/gtk_client.c` | Login dialog, chat window, To dropdown, priority, users panel |
| CLI Client | `src/client/client.c`, `input_handler.c`, `display.c` | Terminal client |
| Crypto | `src/crypto/ratchet.c`, `aes_utils.c`, `rsa_utils.c`, `crypto_common.c` | Double Ratchet, AES-256-CBC, RSA-2048, HKDF |
| TLS | `src/tls/tls_server.c`, `tls_client.c` | TLS 1.3 wrap/unwrap |
| Adaptive Engine | `src/engine/adaptive_engine.c`, `metrics_collector.c` | State machine, rolling metrics |
| Transport | `src/transport/multipath.c`, `offline_queue.c`, `priority_queue.c` | Dual-path, offline storage, priority send |
| IDS | `src/security/intrusion.c` | Per-IP blocking, replay detection |
| Net | `src/net/socket_utils.c`, `dns_resolver.c`, `udp_notify.c` | Sockets, DNS, UDP |

---

## 6. Wire Protocol

Every message — from DH key exchange to chat to user-list pushes — uses the same 28-byte binary header:

```
version(1) | msg_type(1) | priority(1) | flags(1) | msg_id(16) | payload_len(4) | crc32(4)
```

- `msg_id`: 128-bit random ID per message, used for deduplication across TCP and UDP paths
- `priority`: `NORMAL(0)`, `URGENT(1)`, `CRITICAL(2)` — propagated end-to-end
- `flags` bit 1 (`MSG_FLAG_IS_OFFLINE_REPLAY`): marks replayed offline messages so client shows `[queued]` badge
- All `MSG_CHAT` payloads are **constant 4112 bytes** (16-byte IV + 4096-byte AES ciphertext) — traffic analysis resistance

**Message types:**

| Type | Value | Purpose |
|------|-------|---------|
| MSG_DH_INIT | 0x01 | Client sends X25519 public key |
| MSG_DH_RESP | 0x02 | Server sends X25519 public key |
| MSG_AUTH_REQ | 0x03 | Username + RSA pubkey PEM + RSA signature |
| MSG_AUTH_OK | 0x04 | Auth accepted |
| MSG_AUTH_FAIL | 0x05 | Auth rejected |
| MSG_CHAT | 0x06 | IV + AES-256-CBC ciphertext (constant 4112 B) |
| MSG_RATCHET_DH | 0x0C | New X25519 pubkey for DH ratchet step |
| MSG_OFFLINE_STORED | 0x0D | Server confirms offline queue write |
| MSG_ENGINE_STATE | 0x0F | Server broadcasts current adaptive mode |
| MSG_USER_LIST_REQ | 0x10 | Client requests online user list |
| MSG_USER_LIST_RESP | 0x11 | Comma-separated list of online users |
| MSG_ERROR | 0xFF | Human-readable error string |

---

## 7. Connection Lifecycle

Full sequence from TCP connect to active messaging:

```
CLIENT                                      SERVER (handle_client thread)
  |=== TCP connect ========================> |
  |=== TLS 1.3 handshake =================> |
  |--- MSG_DH_INIT  (X25519 pubkey) -------> | Server generates own X25519 keypair
  |<-- MSG_DH_RESP  (X25519 pubkey) -------- |
  |    Both: ECDH → shared_secret            |
  |    Both: HKDF(shared_secret) →           |
  |      root_key, send_chain_key, recv_chain_key
  |    ratchet_init() on both ends           |
  |--- MSG_AUTH_REQ (username + PEM pubkey   |
  |                  + RSA sig) -----------> | verify RSA sig; check duplicate username
  |<-- MSG_AUTH_OK (or MSG_ERROR/FAIL) ----- |
  |    Server: client_table_add()            |
  |<-- MSG_USER_LIST_RESP (broadcast all) -- |
  |    Server: queue_drain() (offline msgs)  |
  |--- MSG_CHAT (IV + ciphertext) ---------> | decrypt with sender ratchet
  |                                          | re-encrypt with recipient ratchet
  |                                          | forward MSG_CHAT to recipient
  |                                          | if offline: queue_store() + MSG_OFFLINE_STORED
  | Every N messages:                        |
  |--- MSG_RATCHET_DH (new X25519 pubkey) -> | ratchet_dh_step() on both sides
  | On disconnect:                           |
  |    Server: client_table_remove()         |
  |<-- MSG_USER_LIST_RESP (broadcast all) -- |
```

**Key design decision — server re-encryption:** The server decrypts each `MSG_CHAT` using the sender's ratchet, extracts plaintext, then re-encrypts using the recipient's ratchet before forwarding. This means each recipient gets a message encrypted with their unique ratchet state. The plaintext passes through server memory briefly. The offline queue stores plaintext (not ciphertext) because the recipient's future ratchet key is unknown at queue time — re-encryption happens when the recipient reconnects with a fresh ratchet state.

---

## 8. Cryptographic Architecture

Four-layer stack (bottom to top):

1. **Key Agreement** — X25519 ECDH (`MSG_DH_INIT` / `MSG_DH_RESP`). Shared secret → HKDF → initial ratchet keys
2. **Identity Authentication** — RSA-2048 signatures. `MSG_AUTH_REQ` carries username, PEM-encoded public key, RSA signature over `challenge = username_bytes`. Server stores `data/keys/<username>.pub` on first login (self-registration). Subsequent logins re-verify against stored key
3. **Transport Confidentiality** — TLS 1.3 (`SSL_CTX_set_min_proto_version(TLS1_3_VERSION)`). Wraps TCP socket before any application bytes exchanged. TLS 1.2 and below disabled
4. **E2EE Confidentiality** — AES-256-CBC with per-message keys derived from Double Ratchet chain steps. Operates inside TLS. Even if TLS is compromised, per-message AES keys are protected by the ratchet

**HKDF (RFC 5869) usage:**
- Initial ratchet setup: `HKDF(salt=0, ikm=shared_secret, info="ratchet_root_init")` → root_key
- Chain key advancement: `kdf_ck(chain_key)` → (new_chain_key, message_key)
- DH ratchet step: `kdf_rk(root_key, dh_output)` → (new_root_key, new_chain_key)

All key material wiped immediately after use with `OPENSSL_cleanse()` (not `memset` — compiler cannot optimize it away).

---

## 9. Double Ratchet Protocol

Follows the Signal Double Ratchet specification. Provides **forward secrecy** and **break-in recovery**.

**State structure:**
```c
typedef struct {
    uint8_t  root_key[32];
    uint8_t  send_chain_key[32];   // advances on each sent message
    uint8_t  recv_chain_key[32];   // advances on each received message
    EVP_PKEY *dh_keypair;          // current X25519 ephemeral keypair
    EVP_PKEY *peer_dh_pubkey;      // peer's latest X25519 public key
    uint32_t  send_counter;
    uint32_t  recv_counter;
    uint32_t  prev_send_counter;
} RatchetState;
```

**Initialization** (after X25519 DH, both sides):
```
root_key       = HKDF(salt=0, ikm=shared_secret, info="ratchet_root_init")
send_chain_key = HKDF(salt=0, ikm=shared_secret, info="ratchet_send_init")  // initiator
recv_chain_key = HKDF(salt=0, ikm=shared_secret, info="ratchet_recv_init")  // initiator
// responder swaps send/recv info strings so chains align
```

**Symmetric ratchet (KDF_CK) — per-message forward secrecy:**
```
msg_key       = HMAC-SHA256(chain_key, 0x01)   // 32 bytes → used as AES-256 key
new_chain_key = HMAC-SHA256(chain_key, 0x02)   // replaces chain_key; old discarded
```
Each message uses a unique derived key. Compromising one message key reveals nothing about past or future messages.

**DH ratchet (KDF_RK) — break-in recovery:**
Triggered every `dh_ratchet_freq` messages (10 in NORMAL mode, 1 in HIGH_RISK mode):
```
dh_output        = ECDH(our_keypair, peer_new_pubkey)
new_root_key, new_chain_key = HKDF(salt=root_key, ikm=dh_output, ...)
```
New X25519 ephemeral keypair generated. New public key sent as `MSG_RATCHET_DH`. Even if an attacker records ciphertext and later compromises a chain key, they cannot derive message keys from before or after the DH step.

**Message encryption flow:**
```
msg_key  = ratchet_send_step()    → 32-byte key
iv       = RAND_bytes(16)         → fresh random IV per message
padded   = zero-pad(plaintext, 4096 bytes)
cipher   = AES-256-CBC(key=msg_key, iv=iv, plaintext=padded)
payload  = iv(16 B) || cipher(4096 B)  → constant 4112 bytes
```

**Key lifetime:**

| Key | Lifetime |
|-----|----------|
| `msg_key` | Single message — zeroed with `OPENSSL_cleanse()` immediately after use |
| `send_chain_key` / `recv_chain_key` | Until next symmetric ratchet step |
| `root_key` | Until next DH ratchet step |
| `dh_keypair` | Until next DH ratchet step |

**State persistence:** `ratchet_serialize()` writes keys and counters to a buffer, encrypted with a passphrase-derived AES key, stored at `~/.aschat/<username>.ratchet` (mode `0600`). The DH private key is NOT serialized — on deserialization a fresh keypair is generated and `MSG_RATCHET_DH` is sent immediately to re-synchronize.

---

## 10. Adaptive Engine

Background thread (1-second evaluation tick). Reads metrics, evaluates thresholds, updates shared `EngineState` read by all other modules.

**Three states and their transport/crypto parameters:**

| Parameter | NORMAL | UNSTABLE | HIGH_RISK |
|-----------|--------|----------|-----------|
| `max_retries` | 3 | 7 | 10 |
| `retry_delay_ms` | 100 ms | 200 ms | 300 ms |
| `chunk_size` | 4096 B | 512 B | 256 B |
| `use_udp_backup` | yes | yes | yes |
| `force_padding` | no | no | **yes** |
| `random_delay` | no | no | **yes** (100–500 ms random) |
| `dh_ratchet_freq` | every 10 msgs | every 10 msgs | **every 1 msg** |

**Transition logic:**
```
if (auth_fail_count >= 5 OR replay_count >= 3 OR packet_loss >= 20%)
    → MODE_HIGH_RISK   (immediate)

else if (packet_loss >= 5% OR consecutive_timeouts >= 3)
    → MODE_UNSTABLE    (immediate)

else if (metrics stable for 30 seconds)
    → MODE_NORMAL      (hysteresis — prevents thrashing)
```

Escalation is **immediate**. Downgrade requires **30 seconds of stable metrics**.

**Why HIGH_RISK is different:**
- `force_padding=1` — all messages padded to 4096 bytes regardless of content, hiding message length
- `random_delay=1` — 100–500 ms random delays between retries, frustrating timing-based traffic analysis
- `dh_ratchet_freq=1` — new DH key every single message, maximum forward secrecy and break-in recovery rate

**Metrics structure:**
```c
typedef struct {
    float    packet_loss_rate;       // rolling average over last 100 sends
    uint32_t rtt_ms;                 // smoothed RTT
    uint32_t auth_fail_count;        // auth failures since last reset
    uint32_t replay_count;           // replay attacks detected
    uint32_t consecutive_timeouts;
} Metrics;
```

The IDS subsystem calls `metrics_record_auth_fail()` and `metrics_record_replay()` directly — security events and network events feed into the same engine.

---

## 11. Multipath Transport

Every outgoing message is sent over both TCP (via TLS) and UDP simultaneously.

**Send logic:**
```
for (attempt = 0; attempt < engine->max_retries; attempt++) {
    tls_send(ssl, payload, len)           // primary TCP path
    udp_send(udp_fd, udp_dest, payload)  // backup UDP path
    if engine->random_delay: sleep(rand(100, 500) ms)
    else: sleep(engine->retry_delay_ms)
}
// PRIORITY_CRITICAL gets +2 extra retries on top of max_retries
```

**Deduplication ring buffer** (since both TCP and UDP may deliver the same packet):
- 1024-slot ring buffer (`dedup_buf[1024][16]`) keyed on 128-bit message IDs
- `dedup_check(msg_id)`: scans window; if seen, discard silently
- `dedup_add(msg_id)`: insert at index, advance modulo 1024
- Protected by `pthread_mutex_t dedup_lock`
- O(1024) = O(1) at typical chat message rates

---

## 12. Offline Message Queue

For messages sent to a disconnected recipient.

**Storage layout:**
```
data/offline_queue/
  <username>/
    <timestamp_ms>_<msg_id_hex>    ← one file per message
```

- Lexicographic filename sort = chronological order; `queue_drain()` sorts before replay
- Stores **plaintext** (not ciphertext) — recipient's future ratchet key unknown at queue time; re-encryption happens on reconnect
- On reconnect: each stored message re-encrypted with the recipient's current ratchet state, sent as `MSG_CHAT` with `MSG_FLAG_IS_OFFLINE_REPLAY` flag, so client shows `[queued]` badge
- Capacity: 500 messages per user (`OFFLINE_QUEUE_MAX`)
- Security: directories `0700`, files `0600`; username validated against `[a-zA-Z0-9_-]` to prevent directory traversal

---

## 13. Priority Queue

Three-level priority queue on the client send path:

| Level | Value | Behavior |
|-------|-------|----------|
| `PRIORITY_NORMAL` | 0 | Standard FIFO |
| `PRIORITY_URGENT` | 1 | Enqueued ahead of NORMAL |
| `PRIORITY_CRITICAL` | 2 | Enqueued at front; send thread wakes immediately; +2 extra retries in multipath |

Priority encoded in `MsgHeader.priority` field and propagated through routing.

---

## 14. Intrusion Detection System

Lightweight per-IP counters with automatic blocking.

**Data structure:**
```c
typedef struct {
    char   ip[48];
    int    auth_fail_count;
    int    blocked;
    time_t blocked_at;
} IpRecord;
// Table of 256 entries (MAX_BLOCKED_IPS)
```

**Block logic:**
- `ids_record_auth_fail(ip, metrics)`: increments counter; if ≥ 5, blocks IP and calls `metrics_record_auth_fail()` to escalate engine to HIGH_RISK
- `ids_record_replay(ip, metrics)`: calls `metrics_record_replay()` — replay detections also escalate engine
- `ids_is_blocked(ip)`: checked in `accept()` loop — blocked IPs never get a TLS handshake
- `ids_expire_blocks()`: passive expiry, called after each message; no timer thread needed
- Block duration: 300 seconds

**IDS log format:**
```
[IDS 2026-06-10 14:23:01] BLOCK from 192.168.1.50
[IDS 2026-06-10 14:28:01] UNBLOCK 192.168.1.50
```

---

## 15. Security Properties

| Property | Mechanism |
|----------|-----------|
| Channel confidentiality | TLS 1.3 |
| E2EE confidentiality | AES-256-CBC with per-message ratchet-derived keys |
| Forward secrecy | Double Ratchet symmetric ratchet — old chain keys discarded after each step |
| Break-in recovery | Double Ratchet DH ratchet step — new X25519 key injects fresh DH entropy |
| Identity authentication | RSA-2048 signature over username, verified server-side |
| Traffic analysis resistance | All payloads padded to constant 4096 bytes before AES encryption |
| Replay attack prevention | 1024-entry dedup ring buffer per connection |
| Timing analysis resistance | Random retry delays (100–500 ms) in HIGH_RISK mode |
| Brute-force / DoS mitigation | IDS per-IP block after 5 auth failures (5-minute block) |
| Key erasure | `OPENSSL_cleanse()` on all key material immediately after use |
| Username uniqueness | Server rejects duplicate usernames with `MSG_ERROR` |
| Adaptive threat response | 3-state engine driven by loss/auth/replay metrics |

---

## 16. Limitations (Must Be Acknowledged in Paper)

1. **Server is a trusted relay** — the server decrypts each `MSG_CHAT` to route and re-encrypt for the recipient. Plaintext passes through server memory. This is not pure E2EE in the Signal sense. A production system would use X3DH pre-key exchange so the server never sees plaintext.

2. **Offline queue stores plaintext** — a server breach exposes queued messages awaiting delivery.

3. **Self-signed TLS certificates** — appropriate for demo/lab; production requires a PKI.

4. **RSA keypairs stored as files** (`data/keys/`) — not hardware-backed; private key security is OS-level file permissions only.

5. **No group E2EE** — broadcast is a loop of directed per-client encryptions, not a proper group ratchet (e.g., MLS / RFC 9420).

6. **No pre-key bundle (X3DH)** — initial DH exchange requires both parties to be online simultaneously (not an issue in practice since the server mediates, but architecturally limits asynchronous session initiation).

---

## 17. Key Constants (from `include/common.h`)

| Constant | Value |
|----------|-------|
| `SERVER_PORT` | 8080 |
| `UDP_PORT` | 8081 |
| `MAX_CLIENTS` | 50 |
| `MSG_PADDED_SIZE` | 4096 bytes |
| `AES_KEY_LEN` | 32 bytes |
| `RSA_KEY_BITS` | 2048 |
| `MSG_ID_LEN` | 16 bytes (128-bit) |
| `DEDUP_WINDOW` | 1024 slots |
| `OFFLINE_QUEUE_MAX` | 500 messages/user |
| `LOSS_THRESHOLD_UNSTABLE` | 5% packet loss |
| `LOSS_THRESHOLD_HIGH_RISK` | 20% packet loss |
| `AUTH_FAIL_THRESHOLD` | 5 failures |
| `REPLAY_THRESHOLD` | 3 detections |
| `BLOCK_DURATION_SEC` | 300 seconds (5 min) |
| `MAX_BLOCKED_IPS` | 256 |

---

## 18. Test Results

| Test | Pass Condition | Result |
|------|---------------|--------|
| Ratchet key uniqueness | 100 derived msg keys all distinct | Pass |
| Forward secrecy | Delete state at msg 50; old msgs unrecoverable | Pass |
| Break-in recovery | Expose chain_key at msg 50; msgs after DH step unreadable | Pass |
| AES roundtrip | Encrypt then decrypt recovers exact plaintext | Pass |
| Constant payload size | All MSG_CHAT = 4112 bytes (verified via tcpdump) | Pass |
| Multipath dedup | Same msg_id via TCP + UDP processed only once | Pass |
| Offline delivery | Messages delivered in chronological order on reconnect | Pass |
| NORMAL → UNSTABLE | Inject 6% packet loss | Transition confirmed |
| UNSTABLE → HIGH_RISK | Inject 21% packet loss | Transition confirmed |
| Any → HIGH_RISK | Inject 5 auth failures | Immediate transition confirmed |
| HIGH_RISK → NORMAL | 30s stable metrics after HIGH_RISK | Hysteresis confirmed |
| IDS block | 5 auth failures from same IP | IP blocked; TLS handshake refused |
| IDS unblock | 300s elapsed | IP unblocked |

---

## 19. References to Cite

- Signal Double Ratchet Algorithm — Marlinspike & Perrin, 2016 (signal.org/docs/specifications/doubleratchet/)
- X3DH Key Agreement Protocol — Marlinspike & Perrin, 2016 (signal.org/docs/specifications/x3dh/)
- TLS 1.3 — RFC 8446, Rescorla, 2018
- HKDF — RFC 5869, Krawczyk & Eronen, 2010
- Messaging Layer Security (MLS) — RFC 9420, Barnes et al., 2023
- Formal security analysis of Signal — Cohn-Gordon et al., IEEE EuroS&P 2017
