# Boundless libcoap native patches

Retain the supplied Runtime I/O and persistence hooks on the current native
baseline. Backport b710ccdd542c3d8d2f5b6c40d59d1938fcb208d3 to check bytes
before decoding extended token lengths. Adapt 2e995218f356d63070afee0517f3c89dd44a944e
so nonempty responses match MID and token; empty ACK/RST remain MID-only.
Preserve the runtime_replay OSCORE authentication-before-removal sequence.
Do not import newer queue, address or locking changes. Native PDU reset and
sendqueue token regressions pass with the complete CUnit suite on macOS arm64.

## Streamed bodies that cannot complete (2026-10-03)

When blocks go to the application as they arrive (no single-body assembly),
libcoap must not restart a body whose ETag changed: the restart duplicates the
original request's method and options without its payload and sends it unasked,
and would splice a new representation onto blocks already delivered. Upstream
`develop` still restarts. The abandonment paths of a streamed Block2 body (ETag
change, a missing ETag, Content-Format change, block tracking overflow, and a
continuation request that cannot be built or sent) instead report
`COAP_NACK_BODY_INCOMPLETE` to the nack handler with the request's own token,
acknowledge the block without passing it on, and request nothing more. A
non-observing exchange is released; an observation keeps its state for its
next notification. Single-body mode keeps upstream behavior. Regressions in
libcoap-rs drive the native client through supplied I/O:
`a_streamed_body_that_cannot_complete_fails_its_own_request_without_another`
(ETag change, Content-Format change, missing ETag; without this patch the restart
is sent) and `a_body_whose_blocks_outrun_tracking_fails_its_own_request_without_another`
(block tracking overflow; without it the block is delivered and nothing reported).
Both fail without the patch and pass with it on macOS arm64. A continuation
request that cannot be built fails only on allocation; one that cannot be sent
through supplied I/O stops the whole client first, so neither has a case of its
own. The complete CUnit suite (177 tests, 1387 assertions; it has no block
receive case) passes with OpenSSL DTLS, TCP and OSCORE on macOS arm64.

## OSCORE ID Context presence

RFC 8613 distinguishes an absent ID Context (CBOR nil in KDF info) from a
present empty byte string. Preserve that distinction in `compose_info`, the
compressed option encoder, and `oscore_find_context`: an explicitly empty KID
Context matches only a configured empty context. An omitted wire KID Context
remains optional for context lookup, including when a context is provisioned.
The existing quoted hex configuration accepts empty sender, recipient and
context IDs; callers must quote empty values.

`t_oscore_context_presence` exercises configuration, an independently calculated
HKDF-SHA-256 key, context selection and option roundtrip. On macOS arm64 with
OpenSSL, its six failing assertions on the previous pin all pass with this patch;
the complete CUnit suite passes (178 tests, 1406 assertions), including unchanged
RFC 8613 Appendix C vectors.

## Supplied-I/O TCP receive maximum (W11d)

A supplied-I/O client checks incoming reliable PDU lengths against its runtime
maximum. Applying the compile-time default as well rejected messages above
8 MiB + 256 bytes even when the embedding application allowed them. Clients
without supplied I/O retain the compile-time guard. The libcoap-rs regression
`supplied_tcp_pdu_uses_the_runtime_receive_maximum` supplies an independently
encoded TCP response above that default within the runtime maximum, and an
oversized response that must be refused before delivery.

## Received Reset target (W5-D7)

Expose an actual received datagram Reset before probe flags, the sendqueue or
blockwise state can change. The read-only classifier shares the queue match
with removal and the application-token resolver with `coap_check_update_token`;
internal ping, Q-Block and extended-token probes never name an application
request. A client-disabled build resolves a queued PDU's own token through the
no-op `coap_block_app_token` fallback, matching its unchanged token update.

`t_runtime_rst_classify` checks application, unmatched and all three probe
targets without changing the queue or probe state. The complete CUnit suite
passes on macOS arm64 with OpenSSL, TCP, OSCORE and Q-Block: 179 tests and 1432
assertions. The controlled-binding regressions
`a_reset_names_the_request_it_answers_before_its_nack`,
`a_reset_of_a_later_block2_block_names_the_application_token`,
`a_reset_of_a_later_block1_block_names_the_application_token` and
`an_unknown_reset_names_no_request` exercise the callback ordering and native
blockwise token restoration through supplied I/O.


## Restore native CSM fallback for supplied I/O (F8)

Restore upstream 851533c3 `src/coap_net.c:1263-1270` verbatim at the
coap_client_delay_first timeout checkpoint: mark csm_not_seen and connect
with the native defaults. The supplied-I/O patch had added a disconnect /
NOT_DELIVERABLE branch here without a recorded requirement; it is removed.
The supplied read/write hooks and their allocation bounds remain required,
but they do not require a different CSM timeout policy. No new timer or
fallback mechanism is introduced.
`t_supplied_csm_timeout_keeps_native_fallback` sends a real pre-CSM request
through supplied I/O with extended-token negotiation, withholds the peer CSM,
and checks the established state plus csm_not_seen after the native timeout.

On macOS arm64 with OpenSSL 3, the CMake/CUnit testdriver passes all 180 tests
(1,440 assertions), including the supplied-I/O CSM-timeout regression.

## Received reliable ending messages (W5-D7 remainder)

Upstream libcoap `851533c3`, `src/coap_net.c::handle_signaling`, disconnects
Release/Abort with COAP_NACK_RST without exposing the received PDU through its
public response, NACK or event handlers. `d23c5888` is fork history, not that
upstream basis. Extend the existing passive received hook with Session target
before the unchanged disconnect. Existing targets retain their numeric values.
CSM, Ping/Pong, token resolution, cancellation and close behavior are unchanged.

`t_received_signaling_precedes_disconnect` supplies independently encoded TCP
Release and Abort with no pending requests: the hook sees actual fields while
the session is established, and the session then disconnects. The binding's
`received_signaling_keeps_fields_and_peer_before_native_disconnect` exercises
that decoder with pending requests and checks received-before-NACK delivery.
Validation: macOS arm64/OpenSSL 3, CMake/CUnit focused test passes both
messages (34 assertions). Removing only this hook makes the same test fail;
restoring it passes again. The CSM and the ending PDU are two actual ingress
polls. The test runner uses CUnit's assertion-failure count as its exit status.
The binding's runtime-I/O checkpoint/consumer regressions pass (21 tests).
