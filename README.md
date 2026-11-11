# sip-nv

SIP, the Session Initiation Protocol
([RFC 3261](https://www.rfc-editor.org/rfc/rfc3261)), is the text protocol that
sets up, changes and ends calls between phones, gateways and servers. A request
such as INVITE goes to a peer, answers come back with a status code, and a
call lives as a dialog until one end sends BYE. This package brings the
protocol to novo-lang as messages and state machines that a caller drives
with bytes it moves itself and a clock it reads itself.

## What it is

A program that takes part in a SIP call needs six things: to read and write
messages, including the awkward ones that real peers send; to run each request
and its answers through the transaction machines that make a lossy datagram
transport reliable; to keep the state of a dialog and form the requests inside
it; to answer a digest challenge; to carry the session description in a body;
and to put these together as an agent that registers, places a call, answers a
call and hangs up. This package is those six.

It has no socket, no clock and no source of randomness. A program that uses it
gives `ua_receive` the octets of each message that arrives, sends the octets of
each `UA_SEND` event, calls `ua_tick` when `ua_next_deadline` says to, and
passes the seed from which tags, branches and Call-IDs are made.

**A message is a value.** `SipMessage` holds the start line, the headers in
order and the body as `Bytes`. A header that is a comma-separated list (Via,
Route, Record-Route, Contact and the others in section 7.3) is one entry for
each element. The writer and the reader are inverses on every message the
reader accepts.

**The reader is strict where the RFC is and not where it is not.** It keeps the
messages of RFC 4475 section 3.1.1 and refuses the ones the RFC says to refuse;
`validate` carries the checks of section 8, so that a parser can be used on its
own.

## Install

```
novo pkg add sip-nv
```

## Example

Two agents, Alice and Bob, are joined by a function that hands the messages of
one to the other. Alice calls Bob with an offer, Bob's agent answers 100 Trying
by itself and reports the call, Bob accepts it, Alice's agent acknowledges it by
itself, and Alice hangs up.

```novo
use std.bytes
use std.str
use sipmsg
use sipua

// Hand every message in `evs` to `to` and answer what `to` says in return,
// leaving out the messages themselves.
fn carry(var to: SipUa, evs: [SipEvent], now: Int) -> [SipEvent]
    var said: [SipEvent] = []
    for e in evs
        if e.kind == sipua.UA_SEND
            for x in sipua.ua_receive(var to, e.data, now)
                list.push(said, x)
    said

fn main() [io]
    var alice = sipua.ua_new("sip:alice@atlanta.example", "sip:alice@192.0.2.1:5060", "sip:atlanta.example",
                             "192.0.2.1", 5060, "UDP", "", "secret", "a")
    var bob = sipua.ua_new("sip:bob@biloxi.example", "sip:bob@192.0.2.4:5060", "sip:biloxi.example",
                           "192.0.2.4", 5060, "UDP", "", "pw", "b")

    // Alice calls Bob with an offer; Bob's agent answers 100 Trying by itself.
    let invite = sipua.ua_invite(var alice, "sip:bob@biloxi.example", "v=0\r\ns=offer\r\n", 0)
    println(sipmsg.parse(invite[0].data)!.method)   // INVITE
    let heard = carry(var bob, invite, 0)
    let incoming = heard[1]
    println("${incoming.kind == sipua.UA_INCOMING} ${incoming.text}")   // true sip:alice@atlanta.example

    // Bob accepts with an answer; Alice's agent acknowledges it by itself.
    let answer = sipua.ua_answer(var bob, incoming.call, 200, "v=0\r\ns=answer\r\n", 100)
    let back = carry(var alice, answer, 100)
    println("${back[1].kind == sipua.UA_ANSWERED} ${back[0].kind == sipua.UA_SEND}")   // true true
    println(sipmsg.parse(back[0].data)!.method)   // ACK
    let established = carry(var bob, back, 100)
    println("${established[0].kind == sipua.UA_CALL_ESTABLISHED}")   // true

    // Alice hangs up: a BYE, answered with a 200.
    let bye = sipua.ua_hangup(var alice, 0, 5000)
    let over = carry(var bob, bye, 5000)
    println("${over[1].kind == sipua.UA_CALL_ENDED} ${bob.calls[0].state == sipua.CALL_ENDED}")   // true true
```

Build and test with `novo pkg build` and `novo test tests`.

## What the package contains

| Module | Contents |
| --- | --- |
| `sipua` | The user agent: `ua_new`, `ua_register`, `ua_invite`, `ua_answer`, `ua_hangup`, `ua_receive` and `ua_receive_from` for what arrives, `ua_tick` and `ua_next_deadline` for time. It runs the transactions, keeps the dialogs, repeats the 2xx of an incoming call, answers a 401 or 407 once, and refreshes a registration at four fifths of the time granted. |
| `siptxn` | The transactions of section 17: client and server, INVITE and other, with timers A to K, the Accepted state of RFC 6026 and the matching rules of sections 17.1.3 and 17.2.3. |
| `sipdialog` | A dialog of section 12: its identity, route set, remote target and CSeq numbers; the requests in it, the ACK of a 2xx, loose and strict routing. |
| `sipbuild` | Forming: a request (section 8.1.1), a response (8.2.6.2), the ACK of a failure (17.1.1.3), a CANCEL (9.1), and the status a server answers before it reads a request. |
| `sipmsg` | The message of section 7: reading and writing, framing on a stream, the long and compact header names, Via with RFC 3581 `rport` and `received`, CSeq, and `validate`. |
| `sipuri` | SIP and SIPS URIs with parameters and headers, equality by section 19.1.4, and name-addr and addr-spec values. |
| `sipauth` | Digest authentication of RFC 7616 and RFC 8760: MD5, SHA-256 and SHA-512-256 with the `-sess` forms, qop `auth`, the nonce count. |
| `sipsdp` | The session description in a body, read and written through `sdp-nv`. |
| `sipstr`, `siperr` | The string helpers of the grammar and the error type. |

Every `pub` declaration is documented where it is declared, and
[the package's page](https://novo-lang.org/packages/sip-nv) is that
reference.

## The rules a user needs

1. **The caller moves the messages.** Nothing reads a socket. `ua_receive` takes
   the octets of one datagram, or one message cut from a stream with
   `frame_length`, and answers a list of `SipEvent`. A message that cannot be
   read is dropped; the agent does not answer it.
2. **Every `UA_SEND` goes to one peer,** the outbound proxy or the registrar. The
   agent does not resolve a name, choose a transport for a URI or follow a
   route to another address. A response is built with the Vias of the request,
   so a caller that serves several peers sends it to `reply_host` and
   `reply_port` of the top Via.
3. **Time is a number the caller measures.** All times are milliseconds on one
   clock. After every call the caller asks `ua_next_deadline` and calls
   `ua_tick` at that time; a tick that is early does nothing, and a late one runs
   every timer that is due.
4. **Randomness is a seed.** `ua_new` takes a string from which every tag, branch,
   Call-ID and cnonce is made with a counter. Two agents that share a seed make
   the same tags. A branch begins with `z9hG4bK`, the magic cookie of section
   8.1.1.7.
5. **The transport decides the timers.** An agent made with UDP runs timers A,
   E and G and waits for D, I, J and K; made with TCP or TLS it does not, and
   those transactions end at once. B, F and H run on both. Over UDP the Via of a
   request carries `rport`.
6. **A call is a number.** `ua_invite` makes call `list.len(ua.calls) - 1`, and an
   incoming call has the number in its `UA_INCOMING` event. Every later event
   about it carries the number.
7. **The 2xx of an incoming call is repeated** from T1, doubling to T2, until the
   ACK comes or 64 T1 have passed, as section 13.3.1.4 says; then the call
   fails. The ACK of a 2xx is sent by the agent and sent again whenever the 2xx
   is repeated.
8. **A challenge is answered once.** A 401 or a 407 to a REGISTER or an INVITE is
   repeated with credentials and a CSeq one higher; a second one is a failure.
   The credentials use qop `auth` when it is offered, and none when no qop is.
9. **Equality of URIs is section 19.1.4's.** `sipuri.equal` undoes escapes in the
   user and password, ignores the case of the host and of the significant
   parameters, and does not equate a missing port with 5060.
10. **A message is read as it is and judged after.** `parse` accepts what the
    grammar accepts, including an unknown scheme in a Request-URI and a request
    with no Max-Forwards (RFC 2543 left it out). `validate` refuses a missing
    To, From, CSeq, Call-ID or Via, a repeated To, From, CSeq, Call-ID or
    Max-Forwards, a CSeq above 2^31 - 1 or with another method than the
    request's, a Max-Forwards above 255, a From, To or Contact that is not an
    address, a Via that cannot be read, a Date that does not end in GMT, and a
    request whose branch is the magic cookie and nothing more.

## What is not included

- **PRACK and 100rel,** RFC 3262: NOT IMPLEMENTED. A provisional response is not
  acknowledged and no RSeq or RAck is read.
- **UPDATE,** RFC 3311: NOT IMPLEMENTED. The agent answers it 405.
- **A re-INVITE or any change inside a call.** An INVITE with a To tag is answered
  481. Session timers, hold and transfer are not here.
- **The `auth-int` quality of protection,** which hashes the body into the digest:
  NOT IMPLEMENTED. A challenge that offers only it is refused. Other algorithms
  than the three named are refused too.
- **The client Accepted state and Timer M of RFC 6026.** The server side of that
  RFC is implemented. A client INVITE transaction ends at the first 2xx, and a
  repeated 2xx is answered with the ACK again by the dialog layer.
- **The matching rules of RFC 2543** for a request whose branch lacks the magic
  cookie: NOT IMPLEMENTED. Such a request has an empty key and is not matched.
- **A proxy, a registrar and any other server.** The agent serves INVITE, ACK,
  CANCEL, BYE and OPTIONS and refuses the rest. `siptxn`, `sipbuild` and
  `sipdialog` are what a server is built from.
- **Forking.** A second dialog made from the same INVITE is not followed.
- **SUBSCRIBE and NOTIFY,** MESSAGE, REFER and the other extensions.
- **DNS** (RFC 3263), **TLS** and **SIP over WebSocket.** A message is bytes; how
  it travels is the caller's.
- **Header grammars past the ones `validate` reads.** A header the package does
  not know is kept as text.

## Tests

The suite is eight files under `tests/`. What they assert comes from the RFCs
rather than from an implementation.

`tests/sip4475_tests.nv` holds the messages of RFC 4475, the SIP torture test
messages, copied from the text of the RFC with the notations it uses expanded
(`<allOneLine>`, `<hex>`, `<repeat>`) and the line ends restored to CRLF: the 25
that sections 3.1.1, 3.3 and 3.4 call valid, which must be read, must pass
`validate` and are checked field by field; the 22 that sections 3.1.2, 3.2.1,
3.3.1, 3.3.8 and 3.3.9 say to refuse, which must be refused by `parse` or by
`validate`; and two the RFC lets an element read or refuse (`escruri` and
`bcast`), which are read. The two with extra spaces in the Request-Line
(`lwsstart`, `trws`) are refused, which the RFC allows.

`tests/sip3261_tests.nv` holds the fourteen messages of the example session of
RFC 3261 section 24.2 (F1 to F14), with the two session descriptions that the RFC
does not show filled in. It checks that each is read and valid, that the
transaction keys pick the right responses, and that the INVITE, the 180, the ACK
and the BYE are formed again, from the library, as the RFC shows them.

`tests/sipbase_tests.nv` holds the string helpers, the URI and name-addr forms,
the equality cases of section 19.1.4, the header operations, the faults of the
start line and of the headers, stream framing, Via, `rport`, CSeq and `validate`.

`tests/siptxn_tests.nv` holds the four machines of section 17 driven with the
clock, with the retransmit times of Table 4 written out: A at 500, 1,500, 3,500
and so on; E and G capped at 4,000; B, F and H at 32,000; D at 32,000; I and K at
5,000; Timer L of RFC 6026. Each runs over UDP and over TCP.

`tests/sipform_tests.nv` holds request and response forming, the ACK of a
failure, the CANCEL, and dialogs: the route set in both orders, loose and strict
routing, the CSeq rules, and every refusal.

`tests/sipauth_tests.nv` holds the MD5 and SHA-256 responses of RFC 7616 section
3.9.1 byte for byte, the SHA-512-256 response (computed with Python's hashlib,
since the RFC gives none), the hash vectors of RFC 1321 and FIPS 180-4, and the
challenge and credentials grammar.

`tests/sipsdp_tests.nv` holds the session description in a body.

`tests/sipua_tests.nv` holds the agent: a registration with a challenge, refresh
and silence; calls between two agents wired in memory, answered, refused,
cancelled, hung up, over UDP and TCP; the lost ACK and the repeated 2xx; an
INVITE nobody answers; the challenge of an INVITE; and the requests an agent does
not serve.

`tests/coverage.sh` merges the line coverage of the suites over `src/` and prints
it; it reads 100%, with no line excused.

## Licence

Apache-2.0. See `LICENSE`.
