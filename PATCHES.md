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
continuation request that cannot be built) instead report
`COAP_NACK_BODY_INCOMPLETE` to the nack handler with the request's own token,
acknowledge the block without passing it on, and request nothing more. A
continuation request that cannot be sent does the same after the block was
passed on (see "A body the application lets go" below). A
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

### A body the application lets go (Boundless #783)

An application that lets a streamed body go (it dropped the body's first
block, released the body, or found a block stale or of a replaced
representation) could not stop libcoap from fetching the rest. The response
handler answers only `COAP_RESPONSE_OK` (ACK) or `COAP_RESPONSE_FAIL` (RST),
and upstream `coap_handle_response_get_block` (`src/coap_block.c`) sends the
next block's request before it returns to `handle_response` (`src/coap_net.c`),
which only then runs the handler. Upstream has no abandonment answer; FAIL
sends an RST, which also deregisters an observation (RFC 7641 section 3.6).

For a streamed body the next block's request is now held on the session
(`runtime_next_block`, `runtime_next_lg_crcv`), and `handle_response` decides it
once the handler has answered, with a third answer, `COAP_RESPONSE_ABANDON`:

- OK sends the held request. If it cannot be sent, the body ends through
  `coap_abandon_streamed_body` with `COAP_NACK_BODY_INCOMPLETE`, after the
  block was passed on.
- FAIL deletes it and sends the RST, as upstream does.
- ABANDON deletes it and ends the body the same way without the NACK, since
  the application asked: the block is acknowledged, an ordinary request's
  block state is released, and an observation or a multicast responder keeps
  its state for its next notification.
- Where no request is held (a single block, a last block, a body that is not
  streamed) ABANDON is OK. A held request no handler answered is deleted with
  its session (`coap_session_mfree`).

The libcoap-rs regressions
`abandoning_an_observation_s_first_block_requests_nothing_and_the_next_notification_arrives`,
`abandoning_a_later_block_of_a_request_requests_nothing_more_and_drops_its_state`,
`abandoning_one_responder_s_body_leaves_another_s_going_on`,
`abandoning_a_stale_first_block_of_an_idle_observation_requests_nothing`,
`accepting_a_block_requests_the_next_once_after_the_answer` and
`rejecting_a_block_resets_it_and_requests_nothing_more` answer each block by
hand through supplied I/O. With the next request sent before the handler
again, as upstream does, all six fail on macOS arm64; with this patch they
pass.

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

## A request that does not fit, typed (Boundless #770)

An embedding application that must tell a program that a request was not sent
because it does not fit needs libcoap's own size decision, not a copy of it: a
copy diverges wherever libcoap normalises the request first (it removes
Q-Block1 without Q-Block support, encodes a Request-Tag from a random counter in
as few bytes as it needs, reserves OSCORE overhead). Upstream refuses a request
that does not fit at three points, each with the bare 0 it also returns for
unrelated failures:

1. `coap_add_token` and `coap_add_option` (`src/coap_pdu.c`) return 0 when
   `coap_pdu_check_resize` cannot grow the PDU past its `max_size`, as they do
   when an allocation fails.
2. `coap_add_data_large_internal` (`src/coap_block.c`) logs "not enough space,
   even the smallest block does not fit (2)" and returns 0 when the room left
   for the body after the token, options, a response's Echo, OSCORE and the
   token pad is below 16 bytes and below the body.
3. The same function logs "(3)" and returns 0 when, after it adds Block1, Size1
   and a Request-Tag, that room is below one 16-byte block.

The same 0 also answers a PDU that already holds data, an OSCORE Proxy-Uri
rebuild that fails, an allocation, and a reliable session before its CSM
(`coap_client_delay_first`). Points 1 to 3 are unchanged.
`coap_add_data_large_runtime_request` is `coap_add_data_large_request` with an
out-parameter, `too_large`, that points 2 and 3 set; every other refusal leaves
it 0. `coap_pdu_runtime_used_size` returns a PDU's `used_size`, so the
application can measure a request it builds without a size bound when point 1
refuses it. Upstream has neither. Neither reports a room: libcoap knows the room
it normalised only at the step that refused.

The libcoap-rs regression
`a_request_is_refused_as_too_large_exactly_where_libcoap_refuses_it` drives each
route (whole body, split body, OSCORE, a short Block1 delta, a Q-Block1 option
libcoap removes, a body over 16 bytes refused at point 2) to libcoap's own
boundary: one option byte more is refused as too large and writes nothing, and
at the boundary the request is sent.

## Upstream ports for blockwise answers (Boundless CoAP stage 1)

Two upstream fixes, ported unchanged ahead of the next-release catch-up:

- `64d5eab7` ("Fix coap_check_send_need_lg_crcv() for requests only"): a NON
  or reliable DELETE answered with Block2 created no `lg_crcv`, and the empty
  ACK branch checked the response instead of the sent request, so the answer
  was dropped and the request never answered. Applied with `git cherry-pick`.
- `592b7bc1` ("Update the lg_crcv for each block"): a Block2 download was
  expired at `MAX_TRANSMIT_WAIT` after its request however fast its blocks
  arrived. The fork's `if (updated_block)` differs from upstream's (no
  `body_free`; the BERT condition), so the three lines were placed by hand at
  the start of that branch, as upstream places them.

Boundless reproduced both before the port (CoAP stage 1 design, runs at
`865feecf`); the binding's suite passes with them.

## Forget a batch of runtime tokens (Boundless CoAP stage 1, F7)

`coap_session_forget_runtime_tokens` replaces the single-token
`coap_session_forget_runtime_token`. Forgetting an in-flight CON removed it from
the send queue and lowered `con_active`, but nothing released the delay queue:
a request delayed behind it (NSTART 1) was sent only when another exchange
completed. Upstream's own token cancellation, `coap_cancel_all_messages`
(`src/coap_net.c`), flushes the delay queue through `coap_session_connected`
after lowering `con_active`; the forget now does the same, once, after every
listed token's queued messages, associations and block state are removed. A
caller ending several requests passes all their tokens, so the release cannot
send a request that is itself being forgotten. The release runs only on an
unreliable session (NSTART governs only UDP and DTLS), so a partly written
stream head is never restarted. A batch whose token matches a partly written
delay-queue head is refused whole, as the single-token call was.

Regressions (libcoap-rs): `forgetting_an_in_flight_con_sends_the_request_delayed_behind_it`
and `forgetting_several_requests_at_once_sends_none_of_them`. Both fail with
the release disabled and pass with it on macOS arm64.

## A Reset frees an NSTART slot only for a CON (Boundless CoAP stage 1)

`coap_dispatch`'s Reset branch lowered `con_active` and flushed the delay queue
before it looked for the reset message. A Reset to a NON, which is never
queued, then freed another request's slot, and a delayed CON was sent while
another CON was still outstanding, against NSTART 1. Boundless sends its
deregistration notice as a NON, so a peer's Reset to it made this routine. The
branch now extracts the node after its existing classification and
`runtime_received` callback, and lowers `con_active` and flushes only when it
removed one: only CONs enter the send queue, as the ACK branch already assumes
(`if (sent && session->con_active)`). Upstream develop (`68849057`) still
decrements before the lookup; the change is proposable upstream.

Regression (libcoap-rs): `a_reset_for_a_non_frees_no_nstart_slot`, with a NON,
an outstanding CON B and a delayed CON C: a Reset for the NON leaves C delayed
until B is answered. It fails without the change and passes with it on macOS
arm64.

## One bodyless deregistration notice under runtime I/O (Boundless CoAP stage 1)

Boundless ends an observation locally at deregistration and forgets its native
state in the same step (owner decision, 2026-11-17). Upstream
`coap_cancel_observe_lkd` re-sends the registration with Observe 1 and the
whole body: a FETCH whose body needs Block1 is uploaded again in an `lg_xmit`
that starts on `obs_token[0]` and goes on with counter tokens from the state
base. That upload outlived the forget (its application token is the Echo token
when the registration was resent with Echo), and a late 2.31 sent the next
cancellation block for an ended observation.

With `context->runtime_read` set, `coap_cancel_observe_lkd` now calls
`runtime_cancel_observe`: one message, the registration's method and options
without its payload and without Block1, Size1 and Q-Block1, with Observe 1, on
`obs_token[obs_token_cnt - 1]` when set, otherwise upstream's own selection
(`obs_token[0]`, else the application token). It is sent with
`coap_send_internal`, as upstream sends its cancellation, and builds no
`lg_xmit`. Without `runtime_read` upstream's code runs unchanged, so ordinary
libcoap clients keep the re-upload. The method follows node-coap (the same
method, no body); go-coap and Californium send GET, which a FETCH-only libcoap
resource answers 4.05 before cancellation is reached.

What this gives up, accepted as best effort (owner decision, 2026-11-17): the
notice's token can miss the one the peer registered (a final Block1 block or a
protected OSCORE registration resent with Echo saves no token; a multi-block
FETCH re-registered through Echo saves its new token at index 0), and a
libcoap server can no longer match a bodyless FETCH through its body-keyed
request cache. The peer then keeps notifying until a Reset or the connection's
end. Upstream's Echo and OSCORE token tracking is untouched, so the catch-up's
fixes there (for example `beb92454`) reach the notice.

Regressions (libcoap-rs) assert the notice on the wire:
`a_get_observation_is_deregistered_by_one_bodyless_non_on_its_token`,
`a_get_observation_resent_with_echo_is_deregistered_on_the_echo_token`,
`an_observation_re_registered_through_echo_on_a_continuation_is_deregistered_on_the_new_token`
(a NON GET established by a Block2 first block, its continuation answered 4.01
Echo, re-registered on a new token) and
`a_multi_block_fetch_observation_is_deregistered_by_one_bodyless_fetch_on_its_final_block_token`
(a three-block FETCH; a late 2.31 continues no upload).
`a_fetch_whose_final_block_was_resent_with_echo_gets_one_bodyless_notice`
asserts one bodyless notice only. Without the patch the two FETCH cases fail
(the body is re-uploaded) and the three GET cases pass, since a single-message
registration's notice equals upstream's but for the body. All pass with it on
macOS arm64; the binding's full runtime-I/O suite (43 tests) and strict Clippy
pass.

## Discharge a CON detached by response dispatch

Upstream v4.3.5b detaches a same-MID NON response's matching CON before
the separate-response path can find it to release credit. Its ACK and Reset
arms release credit directly. Release an actual detached datagram CON once
through the receive-dispatch helper, leaving the unlink itself neutral.
Preserve the detached node for callback correlation and the ordinary
pre-authentication OSCORE path. The supplied runtime's admitted protected
CON/NON paths can also arrive with an already detached node. Token-wide
cancellation still owns nodes left in the queue.

Boundless regression: `separate_non_releases_con_credit_for_the_next_send`
drives same- and different-MID separate NON responses and a Block2
continuation at NSTART=1, then completes a sibling request.

## Admit supplied datagram ACKs and Resets before changing exchanges

Upstream v4.3.5b can unlink an exchange before rejecting a request-code ACK
or invalid-class datagram. It also permits a coded piggybacked ACK through
dispatch when no queued CON matched. The supplied client checks the received
type, code, options and applicable token size first. An ACK needs the
session's peer and a queued CON with the same MID. A coded ACK also needs
the same wire token. An empty ACK keeps its MID-only match. A Reset must
be Empty, including at parse failure, before any Reset classification.
An invalid CON can still receive a Reset, with no request-specific NACK.
A separate response retains its token-based dispatch. The inner OSCORE
datagram goes through the same checks before it can change its association.
Ordinary libcoap dispatch keeps its upstream path.

Boundless #863. The native test
`t_runtime_raw_ack_requires_matching_con` exercises a Block2 ACK without
native block processing. The Boundless regression
`invalid_datagram_replies_leave_the_con_for_its_genuine_answer` checks
malformed ACK, Reset, CON and NON, then the genuine answer and a sibling.

## Initial CON acquires its retransmission holder before the write

Upstream v4.3.5b writes and consumes NSTART credit before allocating the
node for an initial direct datagram CON. Allocation failure returns an
invalid MID and leaves no queued owner for that credit. Prepare the node
and timer after the existing delay checks, before the write. On acceptance,
install the same node. A failed write frees it and its PDU with no credit.
The delay path still allocates or moves its own node, and retransmission
retains the original node. The direct entry also serves the encrypted PDU.

The opt-in test build can refuse the next initial direct CON allocation
on one context. Boundless's `initial_con_without_a_holder_sends_nothing`
checks the Program's send error, absence of the refused request on the wire
and the next CON's successful response with no other packet to free credit.
