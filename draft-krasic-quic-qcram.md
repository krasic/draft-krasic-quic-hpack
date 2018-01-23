---
title: Header Compression for HTTP over QUIC
abbrev: QCRAM
docname: draft-krasic-quic-qcram-latest
date: {DATE}
category: std
ipr: trust200902
area: Transport
workgroup: QUIC

stand_alone: yes
pi: [toc, sortrefs, symrefs, docmapping]

author:
 -
    ins: C. Krasic
    name: Charles 'Buck' Krasic
    org: Google
    email: ckrasic@google.com


--- abstract

The design of the core QUIC transport subsumes many HTTP/2 features, prominent
among them stream multiplexing.  A key advantage of the QUIC transport is stream
multiplexing free of head-of-line (HoL) blocking between streams. In HTTP/2,
multiplexed streams can suffer HoL blocking due to TCP.

If HTTP/2's HPACK is used for header compression, HTTP/QUIC is still vulnerable
to HoL blocking, because of HPACK's assumption of in-order delivery.  This draft
defines QCRAM, a variation of HPACK and mechanisms in the HTTP/QUIC mapping that
allow the flexibility to avoid header-compression-induced HoL blocking.

--- middle

# Introduction

The QUIC transport protocol was designed from the outset to support HTTP
semantics, and its design subsumes many of the features of HTTP/2.  QUIC's
stream multiplexing comes into some conflict with  header compression.  A key
goal of the design of QUIC is to improve stream multiplexing relative to HTTP/2
by eliminating HoL (head of line) blocking, which can occur in HTTP/2.  HoL
blocking can happen because all HTTP/2 streams are multiplexed onto a single TCP
connection with its in-order semantics.  QUIC can maintain independence between
streams because it implements core transport functionality in a fully
stream-aware manner.  However, the HTTP/QUIC mapping is still subject to HoL
blocking if HPACK is used directly.  HPACK exploits multiplexing for greater
compression, shrinking the representation of headers that have appeared earlier
on the same connection.  In the context of QUIC, this imposes a vulnerability to
HoL blocking (see {{hol-example}}).

QUIC is described in {{?QUIC-TRANSPORT=I-D.ietf-quic-transport}}.  The HTTP/QUIC
mapping is described in {{!QUIC-HTTP=I-D.ietf-quic-http}}. For a full
description of HTTP/2, see {{?RFC7540}}. The description of HPACK is
{{!RFC7541}}, with important terminology in Section 1.3.

QCRAM modifies HPACK to allow correctness in the presence of out-of-order
delivery, with flexibility for implementations to balance between resilience
against HoL blocking and optimal compression ratio.  The design goals are to
closely approach the compression ration of HPACK with substantially less
head-of-line blocking under the same loss conditions.

QCRAM is intended to be a relatively non-intrusive extension to HPACK; an
implementation should be easily shared within stacks supporting both HTTP/2 over
(TLS+)TCP and HTTP/QUIC.

## Head-of-Line Blocking in HPACK {#hol-example}

HPACK enables several types of header representations, one of which also adds
the header to a dynamic table of header values.  These values are then available
for reuse in subsequent header blocks simply by referencing the entry number in
the table.

If the packet containing a header is lost, that stream cannot complete header
processing until the packet is retransmitted.  This is unavoidable. However,
other streams which rely on the state created by that packet *also* cannot make
progress. This is the problem which QUIC solves in general, but which is
reintroduced by HPACK when the loss includes a HEADERS frame.

## Avoiding Head-of-Line Blocking in HTTP/QUIC {#overview-hol-avoidance}

In the example above, the second stream contained a reference to data
which might not yet have been processed by the recipient.  Such references
are called "vulnerable," because the loss of a different packet can keep
the reference from being usable.

The encoder can choose on a per-header-block basis whether to favor higher
compression ratio (by permitting vulnerable references) or HoL resilience (by
avoiding them). This is signaled by the BLOCKING flag in HEADERS and
PUSH_PROMISE frames (see {{hq-frames}}).

If a header block contains no vulnerable header fields, BLOCKING MUST be 0.
This implies that the header fields are represented either as references
to dynamic table entries which are known to have been received, or as
Literal header fields (see {{RFC7541}} Section 6.2).

If a header block contains any header field which references dynamic table
state which the peer might not have received yet, the BLOCKING flag MUST be
set.  If the peer does not yet have the appropriate state, such blocks
might not be processed on arrival.

The header block contains a prefix ({{absolute-index}}). This prefix contains
table offset information that establishes total ordering among all headers,
regardless of reordering in the transport (see {{overview-absolute}}).  In
blocking mode, the prefix additionally identifies the minimum state required to
process any vulnerable references in the header block (see `Depends` in Section
{{overview-absolute}}).  When the necessary state has arrived, the header block
can be processed. Notice that while blocked, HB's header field data remains in
stream B's flow control window.


# HTTP over QUIC mapping extensions {#hq-frames}

## HEADERS and PUSH_PROMISE

HEADERS and PUSH_PROMISE frames define a new flag.

BLOCKING (0x01):
: Indicates the stream might need to wait for dependent headers before
  processing.  If 0, the frame can be processed immediately upon receipt.

HEADERS frames can be sent on the Connection Control Stream as well as on
request / push streams.

## HEADER_ACK

The HEADER_ACK frame (type=0x8) is sent from the decoder to the encoder when the
decoder has fully processed a header block on a stream.  It is used by the
encoder to determine whether subsequent indexed representations that might
reference that block are vulnerable to HoL blocking.  The HEADER_ACK frame does
not define any flags, and has no payload.

# HPACK extensions

## Allowed Instructions

HEADERS frames on the Control Streams SHOULD contain only Literal with
Incremental Indexing representations.  Frames on this stream modify the
dynamic table state without generating output to any particular request.

HEADERS and PUSH_PROMISE frames on request and push streams MUST NOT contain
Literal with Incremental Indexing representations.  Frames on these streams
reference the dynamic table in a particular state without modifying it, but emit
the headers for an HTTP request or response.

## Header Block Prefix {#absolute-index}

In HEADERS and PUSH_PROMISE frames, HPACK Header data are prefixed by a pair of
integers: `Fill` and the `Evictions`.

`Fill` is the number of entries in the table, and `Evictions` is the cumulative
number of entries that have been evicted from the table.  Their sum is the
cumulative number of entries inserted before the following header block was
encoded.  Each is encoded as a single 8-bit prefix integer:

~~~~~~~~~~  drawing
    0 1 2 3 4 5 6 7
   +-+-+-+-+-+-+-+-+
   |Fill       (8+)|
   +---------------+
   |Evictions  (8+)|
   +---------------+
~~~~~~~~~~
{:#fig-base-index title="Absolute indexing (BLOCKING=0x0)"}

{{overview-absolute}} describes the role of `Fill` and {{evictions}} covers the
role of `Evictions`.

When the BLOCKING flag is 0x1, a the prefix additionally contains a third HPACK
integer (8-bit prefix) 'Depends':

~~~~~~~~~~  drawing
    0 1 2 3 4 5 6 7
   +-+-+-+-+-+-+-+-+
   |Fill       (8+)|
   +---------------+
   |Evictions  (8+)|
   +---------------+
   |Depends    (8+)|
   +---------------+
~~~~~~~~~~
{:#fig-prefix-long title="Absolute indexing (BLOCKING=0x1)"}

Depends is used to identify header dependencies, namely the largest table entry
referred to by indexed representations within the following header block.  Its
usage is described in {{overview-hol-avoidance}}.  The largest index referenced
is `Evictions + Fill - Depends`.

## Hybrid absolute-relative indexing {#overview-absolute}

HPACK indexed entries refer to an entry by its current position in the dynamic
table.  As Figure 1 of {{!RFC7541}} illustrates, newer entries have smaller
indices, and older entries are evicted first if the table is full.  Under this
scheme, each insertion to the table causes the index of all existing entries to
change (implicitly).  Implicit index updates are acceptable for HTTP/2 because
TCP is totally ordered, but are problematic in the out-of-order context of
QUIC.

QCRAM uses a hybrid absolute-relative indexing approach.  The prefix defined in
{{absolute-index}} is used by the decoder to interpret all subsequent HPACK
instructions at absolute positions for indexed lookups and insertions.

Since QCRAM handles blocking at the header block level, it is an error if the
HPACK decoder encounters an indexed representation that refers to an entry
missing from the table, and the connection MUST be closed with the
`HTTP_HPACK_DECOMPRESSION_FAILED` error code.

## Preventing Eviction Races {#evictions}

Due to out-of-order arrival, QCRAM's eviction algorithm requires changes
(relative to HPACK) to avoid the possibility that an indexed representation is
decoded after the referenced entry has already been evicted.  QCRAM employs a
two-phase eviction algorithm, in which the encoder will not evict entries that
have outstanding (unacknowledged) references.  The QCRAM encoder maintains a
counter as entries are evicted, which is the cumulative number of evictions so
far, `Evictions` ({{absolute-index}}).  On arrival at the decoder, if
`Evictions` is higher than previously seen, the decoder MUST evict the
appropriate number of entries.  Unlike HPACK, where the decoder follows the same
logic as the encoder to perform evictions, in QCRAM the decoder evicts
exclusively based on the encoder's explicit guidance.

### Blocked Evictions

The decoder MUST NOT permit an entry to be evicted while a reference to that entry remains
unacknowledged.  If a new header to be inserted into the dynamic table would cause
the eviction of such an entry, the encoder MUST NOT emit the insert instruction
until the reference has been processed by the decoder and acknowledged.

The encoder can emit a literal representation for the new header in order to
avoid encoding delays, and MAY insert the header into the table later if
desired.

To ensure that the blocked eviction case is rare, references to the oldest
entries in the dynamic table SHOULD be avoided.  When one of the oldest entries
in the table is still actively used for references, the encoder SHOULD emit an
Indexed-Duplicate representation instead (see {{indexed-duplicate}}).

## Refreshing Entries with Duplication {#indexed-duplicate}

~~~~~~~~~~  drawing
    0 1 2 3 4 5 6 7
   +-+-+-+-+-+-+-+-+
   |0|0|1|Index(5+)|
   +-+-+-+---------+
~~~~~~~~~~
{:#fig-index-with-duplication title="Indexed Header Field with Duplication"}

*Indexed-Duplicates* insert a new entry into the dynamic table which duplicates
an existing entry. {{RFC7541}} allows duplicate HPACK table entries, that is
entries that have the same name and value.

This replaces the HPACK instruction for Dynamic Table Size Update (see Section
6.3 of {{RFC7541}}, which is not supported by HTTP over QUIC.

### Mandatory Entry De-duplication {#de-duplication}

To help mitigate memory consumption due to duplicate entries, HPACK for QCRAM is
required to de-duplicate strings in the dynamic table. The table insertion logic
should check if the new entry matches any existing entries (name and value), and
if so, table accounting MUST charge only the overhead portion ({{!RFC7541}}
Section 4.1) to the new entry.

Specific de-duplication mechanisms are left to implementations, but using a map
in conjunction with reference counted pointers to strings would be typical.

# Performance considerations

## Speculative table updates {#speculative-updates}

Implementations can *speculatively* send header frames on the HTTP Control
Streams which are not needed for any current HTTP request or response.  Such
headers could be used strategically to improve performance.  For instance, the
encoder might decide to *refresh* by sending Indexed-Duplicate representations
for popular header fields ({{absolute-index}}), ensuring they have small indices
and hence minimal size on the wire.

## Fixed overhead.

HPACK defines overhead as 32 bytes ({{!RFC7541}} Section 4.1).  QCRAM adds some
per-entry state, to track acknowledgment status and eviction reference count,
and requires mechanisms to de-duplicate strings.  A larger value than 32 might
be more accurate for QCRAM.

## Co-ordinated Packetization

When a dynamic table entry is both defined and referenced by header blocks
within the same packet, there is no risk of HoL blocking and using an indexed
representation is strictly better than using a literal.  An implementation could
attempt to exploit this exception by employing co-ordination between QCRAM
compression and QUIC transport packetization.  However, if the packet is lost,
the transport might choose a different packetization when retransmitting the
missing data.

# Security Considerations

TBD.

# IANA Considerations

This document registers a new frame type, HEADER_ACK, for HTTP/QUIC. This will
need to be added to the IANA Considerations of {{QUIC-HTTP}}.

# Acknowledgments

This draft draws heavily on the text of {{!RFC7541}}.  The indirect input of
those authors is gratefully acknowledged, as well as ideas from:

* Mike Bishop

* Patrick McManus

* Biren Roy

* Alan Frindell

* Ian Swett

* Ryan Hamilton


