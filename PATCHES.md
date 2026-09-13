# Controlled Runtime client

Base: libcoap v4.3.5b (`851533c3cf63d16984d370ce39d586ecb3694971`).
Existing BSD licenses and history are retained.

The embedding application supplies bounded nonblocking stream/datagram I/O.
The native client opens no replacement socket or DNS path in controlled mode.
Read/write, complete DER certificate-chain verification, durable sender reservation
and authenticated replay-window persistence callbacks are exposed in `coap_net.h`.
The caller owns one context on one thread and installs callbacks before sessions.
It must return would-block promptly and must not re-enter native APIs from callbacks.

Native changes:

- Controlled I/O never passes the invalid native descriptor to poll/select.
  Internal handshake waits poll at most 10 ms; missing reliable CSM fails closed.
  Reliable message lengths are bounded before body allocation.
- The application can retire an original token and its native block/Observe,
  delayed/retransmitted and OSCORE state. A partially written reliable packet
  cannot be forgotten independently; the owner must stop that connection.
- BERT streaming honors absence of SINGLE_BODY. The last multi-unit BERT packet
  marks only its last logical block final; it no longer truncates the response.
- Multicast retains a collection template and independent native Block2 state per
  responder. Continuations use that responder's unicast address. Native receive
  state is capped at 256 entries; collection templates end by explicit retirement.
  Skeletal PDU copies relocate their token pointer into their own allocation.
- DTLS verification passes the whole presented certificate chain to the Runtime.
  Native PKI acceptance cannot bypass a rejecting Runtime verifier.
- OSCORE checks sender-reservation failures before using a sequence, persists
  authenticated replay state before plaintext delivery and restores that state
  explicitly. Failed authentication restores the exact prior replay window,
  preserves request associations and does not acknowledge the local send queue.
  A response without a Partial IV may authenticate only once per request, including
  an Observe request; later notifications use the persisted sender replay window.
- Native binary/string allocations are wiped before freeing private key material.

Interoperability is exercised through IMAPipe's safe wrapper against aiocoap
0.4.17, TinyDTLS and independent OpenSSL: UDP/TCP Block1/2, BERT, Observe,
discovery, two-responder multicast Block2, PSK, complete certificate chains,
hostname/trust/mTLS failures, OSCORE reopen, forged tags/IVs and replayed responses.
No server, business workflow, reconnect, execution persistence or alternative
CoAP engine is added to the embedding application.
