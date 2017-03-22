---
title: Header Compression for HTTP over QUIC
abbrev: HPACK
docname: draft-krasic-qpack-latest
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
    role: editor

normative:

  QUIC-TRANSPORT:
    title: "QUIC: A UDP-Based Multiplexed and Secure Transport"
    date: {DATE}
    author:
      -
        ins: J. Iyengar
        name: Jana Iyengar
        org: Google
        role: editor
      -
        ins: M. Thomson
        name: Martin Thomson
        org: Mozilla
        role: editor

  QUIC-HTTP:
    title: "Hypertext Transfer Protocol (HTTP) over QUIC"
    date: {DATE}
    author:
      -
        ins: M. Bishop
        name: Mike Bishop
        org: Microsoft
        role: editor

--- abstract

The design of the core QUIC transport and the mapping of HTTP semantics over it
subsume many HTTP/2 features, prominent among them stream multiplexing and HTTP
header compression.  A key advantage of the QUIC transport is that provides
stream multplexing free of HoL blocking between streams, while in HTTP/2
multiplexed streams can suffer HoL blocking primarily due to HTTP/2's layering
above TCP.  However, assuming HPACK is used for header compression, HTTP over
QUIC is still vulnerable to HoL blocking, because of how HPACK exploits header
redundancies between multiplexed HTTP transactions.  This draft defines QPACK, a
variation of HPACK and mechanisms in QUIC's HTTP mapping that allow QUIC
implementations the flexibility to avoid header-compression induced HoL
blocking.

--- middle

# Introduction

The QUIC transport protocol was designed from the outset to support HTTP
semantics, and its design subsumes most of the features of HTTP/2.  Two of those
features, stream multiplexing and header compression come into some conflict in
QUIC.  A key goal of QUIC's design is to improve stream multiplexing relative to
HTTP/2, by eliminating HoL (head of line) blocking that can occur in HTTP/2.
HoL blocking can happen because HTTP/2 streams are multiplexed onto a single TCP
connection with its in-order semantics.  QUIC can maintain independence between
streams because it implements core transport functionality in a fully
stream-aware manner.  However, the HTTP over QUIC mapping is still subject HoL
blocking if HPACK is used directly as in HTTP/2.  HPACK exploits multiplexing
for greater compression, shrinking the representation of headers that have
appeared earlier on the same connection.  In the context of QUIC, this imposes a
vulnerability to HoL blocking as will be described more below.

QUIC is described in {{QUIC-TRANSPORT}}.  The HTTP over QUIC mapping is decribed
in {{QUIC-HTTP}}. For a full description of HTTP/2, see {{!RFC7540}}.  The
description of HPACK is {{!RFC7541}}.

# QPACK overview

Readers may wish to refer to {{!RFC7540}} Section 1.4 to review HPACK
terminology, and {{QUIC-HTTP}}, Sections 4 on "HTTP over QUIC stream mapping"
and 4.2.1 on "Header Compression".

This draft extends HPACK and the HTTP over QUIC mapping with the *option* to
avoid HoL blocking, in a backward compatible fashion.  QPACK strives to solve
HoL blocking in the simplest way possible. To that end, the mechanisms QPACK
defines are largely at the granularity of header blocks, as opposed to
individual header field representations.

## Example of HoL blocking

The following is an example of how HPACK can induce HoL blocking in QUIC. Assume
two message control streams `A` and `B`, and corresponding header blocks `HA`
and `HB`. Stream `B` experiences HoL blocking due to `A` as follows:

1. HPACK encodes header field `HB[i]` using an index that refers to a table
   entry that resulted from header field `HA[j]`.
2. `HA` and `HB` are delivered via distinct packets that are inflight in the
   same round trip.

3. `HB`'s packet is delivered but `HA`'s is dropped.  HPACK can not decode `HB`
   until `HA`'s packet is successfully retransmitted.

## How QPACK avoids HoL blocking
Continuing the example, QPACK's approach is as follows.

1. `HB[i]` can refer to `HA[j]` if `HA[j]` was delivered in a prior round trip.
2. `HB[i]` can refer to `HA[j]` if `HA` and `HB` are to be delivered in the same
   packet.
3. If QPACK is enabled, `HB[i]` will be represented using an HPACK literal.
   Otherwise an indexed representation may be used, but HB must processed
   in-order, after HA.

It is worth noting that rules 1. and 2. are situations where `HB` is not at risk
of HoL blocking, even without QPACK.  Only in rule 3 does QPACK come into play
giving the encoder the choice between HoL avoidance or better compression.

### Absolute Indexing

HPACK indexed entries refer to an entry by its current position in the dynamic
table.  As Figure 1 of {{!RFC7541}} illustrates, newest entries have smallest
indices, and oldest entries are evicted first if the table is full.  Under this
scheme, each insertion to the table causes the index of all existing entries to
change (implicitly).  The approach is acceptable for HTTP/2 because TCP is
totally ordered, but it is is problematic in the out-of-order context of QUIC.

QPACK uses a hybrid absolute-relative indexing approach.  Every QPACK header
block starts with an integer that conveys the base index, the total number of
entries that had been inserted to the dynamic table before encoding the current
header block.  The format of individual indexed representations does not change,
but their semantics become absolute in combination with the base index.
Similarly, the base index is used to perform table insertions at unambiguous
positions.

# Changes to HPACK and HTTP over QUIC

QPACK is optional on a per header block basis, and signaled by a new flag in the
HEADERS and PUSH_PROMISE frames.  If this flag is absent, then the header block
should be processed in strict order as per Section 4.2.1 of the HTTP mapping.

## HPACK changes

QPACK adds three integer *epochs* to HPACK state, all derived from the sequence
numbers of HTTP Mapping (refer to {{QUIC-HTTP}} Sections 5.2.2 and 5.2.4.), and
provided to the HPACK layer by the HTTP mapping:

1. `encode_epoch`: the sequence number of the frame enclosing the header block,
   as per the HTTP Mapping.  When entries are added to they dynamic table, the
   current encode epoch is stored with the entry.
2. `packet_epoch`: the first encode epoch in the current QUIC packet.  When
   multiple header blocks are packed into a single QUIC packet, the header
   blocks should be ordered.
3. `commit_epoch`: the highest in-order encode epoch acknowledged to the
   encoder side.

The following must hold: `encode_epoch >= packet_epoch > commit_epoch`.
The next section provides more detail of how the values are computed.

### Indexed representations

As each header block is processed, HPACK is informed whether QPACK is enabled.
If so, the encoder will emit an indexed representation only if there is a
matching entry in the dyamic table such that: `entry.encode_epoch <=
commit_epoch or entry.encode_epoch >= packet_epoch`.  Otherwise a literal must
be used.

### Indexing

Every QPACK header block must start with a single HPACK integer that encodes the
value of the base index.  As described above, the decoder will use this as the
starting point for insertions, and for interpreting indexed representations.

## HTTP Mapping changes

HoL avoidance is signalled on a per header block basis, a new flag is reserved
in HEADER frames that signals whether the block is QPACK encoded.  When received
on the decoding side, if the HEADER frame contains the QPACK flag set, then the
header block may be processes immediatly instead of in-order.

When sending headers, the HTTP mapping layer provides the commit, packet, and
encoding epochs to the HPACK layer:

* then encoding epoch increments for every new header encoded.

* the mapping layer keeps track of header blocks by their encode epochs, and
  monitors transport acknolwedgments to determine when `commit_epoch` can
  advance, that is when the highest in-order acknowledged encode epoch has
  increased.  *This piggybacks on existing QUIC transport mechanisms, no
  additional wire format changes are needed.*

* the mapping layer coordinates with packet writing to manange space available
  for header blocks, and advances the packet epoch at packet boundaries.
  *Although sub-optimal, an simpler implementation could ignore packet
  boundaries and hold that `packet epoch == encode epoch`.*

### Table evictions

Since QPACK allows headers to be processed out of order, it might be possible
that an header block may contain references to entries that have already been
evicted by the time it arrives.  For example, suppose HB was encoded after HA,
and HB evicts an entry reference by HA.   If due to network drops HB is decoded
first, the reference in HA will become invalid.

To handle this with minimal complexity, QPACK takes the following approach: if
while encoding the current header block, an eviction becomes necessary, then
QPACK must be disabled for the current header block.  In the above example, HB
could not be QPACK enabled, hence decoding HB must wait for HA to be decoded
first.

*Compared to other QUIC state such as receive buffers, the default table size of
4,096 octets (see {{!RFC7540}} Section 6.5.2.) is very modest.  Deployment data
suggests it is rarely increased in practice, and experiments to increase it did
not yield significant gains.  Consequently, I think it's best to avoid any
heroic measures to deal with performance under full tables. *

# Performance considerations

Beyond sequence numbers already defined in Section 5.2.1 and 5.2.4, the only
additional overhead of QPACK is the base index added to header blocks.  In the
common case, the index should consume 1 byte per header block.

It might be advantageous to allow implemenations to send header frames on
the HTTP control stream (QUIC stream 3).  Such headers would not be associated
with any HTTP transaction, but could be used strategically to improve
performance. For instance, as a means to avoid disabling QPACK because of table
eviction, or to ensure most frequently used entries have the smallest indices.

# Security Considerations

TBD.

# IANA Considerations

This document currently makes no request of IANA, and might not need to.

# Acknowledgements

This draft draws heavily on the text of {{!RFC7541}}.  The indirect input of
those authors is gratefully acknowledged, as well as ideas from:

* Mike Bishop

* Patrick McManus

* Biren Roy


