# WSMP Tech Docs

![WSMP Tech logo](assets/logo.svg)

WSMP (Warp Speed Message Protocol) is a lightweight, UDP-based protocol for fast, reliable messaging between services on high-quality networks (VPC, LAN). It targets payloads up to 64 KB and prioritizes low latency and efficient batching over heavyweight transport features. The reference implementation is built in Rust.

## Why WSMP: Motivation and Advantages over HTTP/2 and QUIC

WSMP was designed to optimize the transmission of small messages in trusted high-speed networks, where traditional protocols like HTTP/2 and QUIC introduce unnecessary overhead, particularly in high-throughput scenarios. Here are the key reasons for creating the protocol:

- Reduced latency for small requests through simplified flow control: both HTTP/2 (over TCP) and QUIC (HTTP/3) use strict credit-based flow control mechanisms. The receiver must actively send WINDOW_UPDATE or MAX_DATA frames to allow the sender to continue transmitting, introducing additional RTT for signaling. In scenarios involving small messages (1–10 KB) at high throughput (thousands of requests per second), this causes latency spikes as the sender may stall waiting for window updates despite available bandwidth. WSMP employs an advisory-driven approach: the receiver only suggests missing fragments or recommends a window size, without mandatory blocking. This eliminates unnecessary signaling overhead and latency, making the protocol ideal for internal microservice communications in data centers or VPCs, where the network is stable and congestion is minimal.
- Avoiding unnecessary congestion control in controlled environments: QUIC and HTTP/2 incorporate built-in congestion control mechanisms (similar to TCP), which are essential in WAN environments with variable latency and loss. In trusted high-speed networks (for example, AWS VPC or on-premises clusters), these mechanisms add superfluous probing and backoff delays that reduce throughput. WSMP relies on sender-driven retries without global congestion signaling, enabling maximum performance without over-engineering.
- Lower overall overhead for tiny payloads: encryption and framing in QUIC/HTTP/2 (with per-packet crypto and multiplexing) increase CPU load and packet size, becoming a bottleneck at speeds over 1 Gbps. WSMP minimizes header size (8–16 bytes baseline) and supports message batching, delivering better p99 latency and throughput in scenarios like in-cluster messaging in Kubernetes.

In summary, WSMP is a targeted alternative for niche use cases where the universality of HTTP/2 and QUIC becomes a performance liability. Benchmarks against gRPC or raw HTTP/2 in low-latency environments are recommended to quantify these advantages.

## Design goals

- Small payloads: up to 64 KB per message.
- Fast, reliable delivery inside a trusted network domain.
- Efficient batching via `sendmmsg`/`recvmmsg` and GSO/GRO.
- Minimal protocol overhead with explicit flow control.
- Lightweight like UDP-DNS, but suitable for generic payloads.

## Protocol overview

WSMP has two layers: an encryption layer and a payload layer. The encryption layer establishes a secure session with a TLS-like handshake, while the payload layer carries application messages.

- Transport: UDP with a custom encryption layer and a TLS-like handshake.
- Messaging model: request-response, with association via client-generated `message_id`.
- Reliability: sender-driven retries when delivery is missing; receiver may advise missing pieces during defragmentation. The client defines retry count and backoff strategy, yielding at-least-once delivery semantics.
- Flow control: advisor-driven rather than aggressive window management.
- Session recovery: receiver can send an advisory packet to re-establish a session when it sees an expired or unknown session (for example, after load-balancer re-routing).
- Keepalive: ping/pong advisory packets for liveness checks and latency sampling.

## Handshake packet format (suggested)

The handshake is TLS/DTLS-inspired but transported over WSMP's own UDP record layer.

Record header:

```text
+---------+------------+-------+-----------------+--------+
| version | record_type| epoch | sequence_number | length |
+---------+------------+-------+-----------------+--------+
| 1 byte  | 1 byte     | 1 byte| 4 bytes         | 2 bytes|
```

Handshake fragment (DTLS-style):

```text
+--------+-----------+------------+------------------+-----------------+
| hs_type| hs_length | hs_msg_seq | fragment_offset  | fragment_length |
+--------+-----------+------------+------------------+-----------------+
| 1 byte | 3 bytes   | 2 bytes    | 3 bytes          | 3 bytes         |
```

Handshake message bodies (minimal):

```text
ClientHello:
+---------------+--------------------+------------------+------------+
| client_random | session_id_hint    | cipher_suites    | extensions |
| 32 bytes      | optional           | list             | list       |

ServerHello:
+---------------+------------------+-----------------------+------------+
| server_random | session_id       | cipher_suite_selected | extensions |
| 32 bytes      | 8 or 16 bytes    |                       | list       |

KeyExchange (optional):
+-----------+----------------------+
| key_share | signature (optional) |
| X25519    | server-auth          |

Finished:
+-------------+
| verify_data |
```

## Message format and framing

- Encrypted payload is built from the handshaked session ID and the encrypted content (typically with a nonce) using AES-256-GCM.
- Encrypted payload size must not exceed 64 KB.
- Payload is split into datagram-sized frames; each frame includes its index and the total frame count.
- Frames match standard MTU sizes (for example, 1400 bytes) and are prefixed with their size to simplify GRO-based decoding.
- The last frame should be padded to the configured size to improve GRO coalescing; the size prefix preserves the true payload length with minimal overhead.
- Receiver buffers frames, reassembles once all are present, then decrypts.
- Decrypted payload layout: message `type` (data or advisory), then `message_id`, followed by the message content to the end of the buffer.

### Packet and payload layout (ASCII)

Encrypted payload (before framing):

```text
+------------------+--------------------------+
| session_id (S)   | AES-256-GCM ciphertext   |
|                  | (nonce + encrypted data) |
+------------------+--------------------------+
<= 64 KB total
```

Frame (one UDP datagram):

```text
+--------------+-----------+-------------+----------------------+
| frame_size   | frame_id  | frame_count | encrypted_payload[..]|
+--------------+-----------+-------------+----------------------+
fits MTU (e.g. ~1400 bytes)
```

Decrypted payload (after reassembly and decryption):

```text
+----------+-------------+-------------------+
| type     | message_id  | message_content   |
| (data/   |             | (rest of buffer)  |
| advisory)|             |                   |
+----------+-------------+-------------------+
```

## Performance features

- Message collection and short-term batching to amortize syscalls.
- GSO buffer usage for efficient packetization.
- GRO-enabled receive paths for fast coalescing on ingress.
