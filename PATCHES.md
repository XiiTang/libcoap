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
