---
title: Header Compression for HTTP over QUIC
abbrev: QCRAM
docname: draft-krasic-quic-qcram-02
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
header compression.  A key advantage of the QUIC transport is it provides
stream multiplexing free of HoL blocking between streams, while in HTTP/2
multiplexed streams can suffer HoL blocking primarily due to HTTP/2's layering
above TCP. However if HPACK is used for header compression, HTTP over
QUIC is still vulnerable to HoL blocking, because of how HPACK exploits header
redundancies between multiplexed HTTP transactions.  This draft defines QCRAM, a
variation of HPACK and mechanisms in the QUIC HTTP mapping that allow QUIC
implementations the flexibility to avoid header-compression induced HoL
blocking.

--- middle

# Introduction

The QUIC transport protocol was designed from the outset to support HTTP
semantics, and its design subsumes most of the features of HTTP/2.  Two of those
features, stream multiplexing and header compression come into some conflict in
QUIC.  A key goal of the design of QUIC is to improve stream multiplexing
relative to HTTP/2, by eliminating HoL (head of line) blocking that can occur in
HTTP/2.  HoL blocking can happen because HTTP/2 streams are multiplexed onto a
single TCP connection with its in-order semantics.  QUIC can maintain
independence between streams because it implements core transport functionality
in a fully stream-aware manner.  However, the HTTP over QUIC mapping is still
subject to HoL blocking if HPACK is used directly as in HTTP/2.  HPACK exploits
multiplexing for greater compression, shrinking the representation of headers
that have appeared earlier on the same connection.  In the context of QUIC, this
imposes a vulnerability to HoL blocking as will be described more below
({{hol-example}}).

QUIC is described in {{QUIC-TRANSPORT}}.  The HTTP over QUIC mapping is
described in {{QUIC-HTTP}}. For a full description of HTTP/2, see {{!RFC7540}}.
The description of HPACK is {{!RFC7541}}.

# QCRAM overview

Readers may wish to refer to {{!RFC7541}} Section 1.3 to review HPACK
terminology, and {{QUIC-HTTP}}, Sections 4 on "HTTP over QUIC stream mapping"
and 4.2.1 on "Header Compression".  QCRAM extensions to HPACK allow correctness
in the presence of out-of-order delivery, with flexibility to balance between
resilience against HoL blocking and compression ratio.

QCRAM is intended to be a relatively non-intrusive extension to HPACK, an
implementation should be easily shared within stacks supporting both HTTP/2 over
(TLS+)TCP and HTTP over QUIC.

## Example of HoL blocking {#hol-example}

The following is an example of how HPACK can induce HoL blocking in QUIC. Assume
two HTTP message exchange streams `A` and `B`, and corresponding header blocks `HA`
and `HB`. Stream `B` experiences HoL blocking due to `A` as follows:

1. HPACK encodes header field `HB[i]` using an index that refers to a table
   entry that resulted from header field `HA[j]`.
2. `HA` and `HB` are delivered via distinct packets that are inflight in the
   same round trip.
3. `HB`'s packet is delivered but `HA`'s is dropped.  HPACK can not decode `HB`
   until `HA`'s packet is successfully retransmitted.

## How QCRAM minimizes HoL blocking {#overview-hol-avoidance}
Continuing the example, QCRAM's approach is as follows.

1. `HB[i]` will not introduce HoL blocking if `HA[j]` was delivered in a prior
   round trip.  To identify this case, QCRAM assumes that QUIC transport
   surfaces acknowledgement notifications to the HTTP layer, and that the QCRAM
   encoder can rely that acknowledged headers have been received by the decoder.

2. `HB[i]` may be represented with one of the Literal variants (see {{RFC7541}}
    Section 6.2), trading lower compression ratio for HoL resiliance.
    
3. `HB[i]` may be represented with an Indexed Representation.  This favors
    compression ratio, but the decoder MUST ensure that HB is not decoded until
    after HA (see blocking in {{overview-absolute}})).

# HPACK extensions

## Header Block Prefix {#absolute-index}

In HEADERS and PUSH_PROMISE frames, HPACK Header data should be prefixed by a
pair of integers: `Fill` and the `Evictions`. `Fill` is the number of entries in
the table, and `Evictions` is the cumulative number entries that have been
evicted from the table.  Their sum is the cumulative number of entries inserted.
Each is encoded as a single HPACK integer (8-bit prefix):

~~~~~~~~~~  drawing
    0 1 2 3 4 5 6 7 
   +-+-+-+-+-+-+-+-+
   |Fill       (8+)|
   +---------------+
   |Evictions  (8+)|
   +---------------+
~~~~~~~~~~
{:#fig-base-index title="Absolute indexing"}

{{overview-absolute}} describes the role of `Fill` and {{evictions}} covers the
role of `Evictions`.

## Hybrid absolute-relative indexing {#overview-absolute}

HPACK indexed entries refer to an entry by its current position in the dynamic
table.
As [Figure 1 of RFC7541](https://tools.ietf.org/html/rfc7541#section-2.3.3)
illustrates, newest entries have smallest indices, and oldest entries are
evicted first if the table is full.  Under this scheme, each insertion to the
table causes the index of all existing entries to change (implicitly).  Implicit
index updates are acceptable for HTTP/2 because TCP is totally ordered, but it
is is problematic in the out-of-order context of QUIC.

QCRAM uses a hybrid absolute-relative indexing approach.  The prefix defined in
{{absolute-index}} is used by the decoder to interpret all subsequent HPACK
instructions at absolute postitions for indexed lookups and insertions. It is 
also used for evictions ({{evictions}}).

As was defined in {{overview-hol-avoidance}} case 3, the encoder has the
option to select indexed representations that are vulnerable to HoL blocking.
Decoder processing of indexed header fields MUST block the encompassing header
block if the referenced entry has not been added to the table yet.

To protect against buggy or malicious peers, a timer should be used to
set an upper bound on such blocking and in treat expiration of the
timer as a decoding error.   However, if the implementation chooses not to abort 
the connection, the remainder of the header block MUST be decoded and output 
discarded.

## Preventing Eviction Races {#evictions}
Due to out of order arrival, QCRAM's eviction algorithm requires changes
(relative to HPACK) to avoid the possiblity that an indexed representation is
decoded after the referenced entry is already evicted.  QCRAM employs a
two-phase eviction algorithm, in which the encoder will not evict entries that
have outstanding (unacknowledged) references.  The QCRAM encoder maintains a
counter as entries are evicted, which is the cumulative number of evictions so
far, `Evictions` ({{absolute-index}}).  On arrival at the decoder, if
`Evictions` is higher than previously seen, the decoder MUST evict all entries
at or below.  Unlike HPACK where the decoder follows the same logic as the
encoder to perform evictions, in QCRAM the decoder evicts exclusively based on
the encoder's explicit guidance.

### Blocked Evictions
In some cases, the encoder must forgo eviction by selecting a literal
representation (blocked eviction), namely in the event that the entry subject to
eviction *is* referenced by one or more unacknowledged header frames. To assure
that the blocked eviction case is rare, a form of thresholding MAY be applied
that constrains selection of Indexed representations, such that the oldest
entries in the dynamic table will largely be evictable.  The constraint is
applied when encoding header fields: comparing the cumulative position (in
bytes) of the matching entry to a threshold, categorizing oldest entries (past
threshold) as at-risk.  Avoiding references to at-risk entries, the
encoder SHOULD use an Indexed-Duplicate representation instead (see
{{indexed-duplicate}}).

## Handling Stream Resets
The QCRAM encoder has the option to select representations that might require
blocking ({{overview-hol-avoidance}} case 3), but the decoder must be
prevented from becoming hung if the stream associated with the referenced entry
is reset.  On stream reset, the QCRAM encoder MUST check if the stream has
unacknowledged headers, and if so resend them on the Control Stream
({{QUIC-HTTP}} Section 4.1).  If header blocks are resent on the control stream,
duplicate arrivals are possible due to reset-acknowledgment races.  The decoder
MUST ignore duplicate header block arrivals, which is straightforward because of
unambiguous indexing (see {{overview-absolute}}).

## Refreshing Entries with Duplication {#indexed-duplicate}

~~~~~~~~~~  drawing
    0 1 2 3 4 5 6 7 
   +-+-+-+-+-+-+-+-+
   |0|0|1|Index(5+)|
   +-+-+-+---------+
~~~~~~~~~~
{:#fig-index-with-duplication title="Indexed Header Field with Duplication"}

*Indexed-Duplicates* are treated as an Indexed Header Field Represention (see
{{!RFC7541}} Section 6.1), additionally inserting a new duplicate entry.
{{RFC7541}} allows duplicate HPACK table entries, that is entries that have the
same name and value.

*Figure 2 annexes the representation for HPACK Dynamic Table Size Update (see
 Section 6.3 of RFC7541), which is not supported by HTTP over QUIC.*

### Manditory Entry De-duplication {#de-duplication}

To help mitigate memory consumption due to duplicate entries, HPACK for QCRAM is
required to de-duplicate strings in the dynamic table. The table insertion logic
should check if the new entry matches any existing entries (name and value), and
if so, table accounting MUST charge only the overhead portion ({{!RFC7541}}
Section 4.1) to the new entry.

Specific de-duplication mechanisms are left to implementations, but using a map
in conjunction with reference counted pointers to strings would be typical.

# Performance considerations

## Speculative table updates {#speculative-updates}
Implementations can *speculatively* send header frames on the HTTP Connection
Control Stream.  Such headers would not be associated with any HTTP transaction,
but could be used strategically to improve performance.  For instance, the
encoder might decide to *refresh* by sending Indexed-Duplicate representations
for popular header fields ({{absolute-index}}), ensuring they have small indices
and hence minimal size on the wire.

## Fixed overhead.
HPACK defines overhead as 32 bytes ({{!RFC7541}} Section 4.1).  QCRAM adds some
per-entry state, to track acknowledgment status and eviction rank, and requires
mechanisms to de-duplicate strings.  A larger value than 32 might be more
accurate for QCRAM.

## Co-ordinated Packetization 
In {{overview-hol-avoidance}} case 3, an exception exists when the
representation of `HA[i]` and `HB[j]` are delivered within the same transport
packet.  If so, there is no risk of HoL blocking and using an indexed
representation is strictly better than using a literal.  An implementation could
exploit this exception by employing co-ordination between QCRAM
compression and QUIC transport packetization.

# Security Considerations

TBD.

# IANA Considerations

This document currently makes no request of IANA, and might not need to.

# Acknowledgments

This draft draws heavily on the text of {{!RFC7541}}.  The indirect input of
those authors is gratefully acknowledged, as well as ideas from:

* Mike Bishop

* Patrick McManus

* Biren Roy

* Alan Frindell

* Ian Swett

* Ryan Hamilton


