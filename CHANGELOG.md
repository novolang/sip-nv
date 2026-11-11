# Changelog

All notable changes to sip-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.0

The first release.

### Added

- **`sipmsg`**, the SIP message of RFC 3261 section 7 read and written as bytes:
  folding, compact header names, list headers split into elements, the body cut
  to Content-Length, stream framing with `frame_length`, Via with the RFC 3581
  `rport` and `received` parameters, CSeq, and `validate`.
- **`sipuri`**, SIP, SIPS and other URIs with parameters and headers, equality by
  section 19.1.4, and name-addr and addr-spec values.
- **`sipbuild`**, request forming (section 8.1), response forming (8.2), the ACK
  of a failure (17.1.1.3), the CANCEL (9.1) and the status a server answers
  before it reads a request.
- **`siptxn`**, the four transaction machines of section 17 with timers A to K
  as caller-driven time, and the Accepted state of the INVITE server transaction
  of RFC 6026.
- **`sipdialog`**, dialogs of section 12 with their route sets, loose and strict
  routing, and the ACK of a 2xx.
- **`sipauth`**, digest authentication of RFC 7616 and RFC 8760: MD5, SHA-256 and
  SHA-512-256, the `-sess` forms, qop `auth` and the nonce count, for the
  challenge and the credentials headers.
- **`sipsdp`**, the session description in a body, through `sdp-nv`.
- **`sipua`**, a user agent that registers, places and answers calls and hangs
  up, as a step over messages the caller moves.
- The messages of RFC 4475 and of RFC 3261 section 24.2 as tests.

### Known

- NOT IMPLEMENTED: PRACK (RFC 3262), UPDATE (RFC 3311), the `auth-int` qop, the
  client Accepted state and Timer M of RFC 6026, the RFC 2543 transaction
  matching, a re-INVITE, forking, and any server role.
- The agent talks to one peer.
