---
###
title: "Support for nothing-new notifications in the DNS"
abbrev: "Nothing-new in the DNS"
category: std
docname: draft-hardaker-dnsop-nothing-new-latest
submissiontype: IETF
consensus: true
v: 3
area: "Operations and Management"
workgroup: "Domain Name System Operations"
keyword:
 - DNS
 - DNSSEC
venue:
  group: "Domain Name System Operations"
  type: "Working Group"
  mail: "dnsop@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/dnsop/"
  github: "https://github.com/hardaker/draft-hardaker-dnsop-nothing-new"

author:
  -
    fullname: Wes Hardaker
    organization: Google, Inc.
    email: ietf@hardakers.net

normative:
  RFC1035:
  RFC6891: # EDNS0

informative:
  BCP237:  # DNSSEC
  RFC8767: # serve stale

--- abstract

The DNS protocol has increasingly needed to carry larger records than
it was originally designed to carry.  This has resulted in performance
impacts due to both the size increases and requiring TCP instead of
only UDP.  Of particular note is the expected large increase in
records relating to post-quantum signing algorithms.  To help
mitigate, but not entirely prevent, these impacts, this document
proposes a new "nothing new" NN flag, a LARGE Redirection Resource
record type, and how these can integrate with current and future
DNSSEC DNSKEY and RRSIG records.

--- middle

# Introduction

(Ed: this is very much a work in progress and is not a fully specified
specification at this point, and as such is not implementable.  It is
designed to promote thought and discussion about how to handle large
requests within the DNS using new mechanisms.)

## Background

The DNS protocol has increasingly needed to carry larger records than
it was originally designed to carry.  This has resulted in performance
impacts due to both the size increases and requiring TCP instead of
only UDP.  Of particular note is the expected large increase in
records relating to Post-Quantum-Computing (PQC) signing algorithms.
Note that while this draft concentrates on PQC algorithms, the
techniques proposed should help mitigate other large packet size
issues with any types of DNS data.

With the increase in size requirements being transmitted over DNS, we
have but a few options to address the need for large RRsets and/or
mitigate the burden on authoritative servers. These are at least some
of the options available:

1. Encourage the switch to TCP for requests which are known to
   generate large responses. Especially those performing DNSSEC (DO
   bit) queries.
   
2. Investigate and deploy DNSSEC signing algorithms and deploy that
   minimize the packet size impacts. We have already done this
   recently, to some extent, with the shift to elliptic curve based
   algorithms in DNSSEC
   
   But PQC algorithms will be significantly larger, even if we
   standardize on an algorithms with the smallest key and signature
   sizes.
   
3. Reduce the need for sending large responses in the first place. The
   most obvious solution to this is to increase TTL values.  However,
   that is not always possible.
   
This draft explores an additional mechanism to solve #3 by further
reducing the quantity of large packets needed to be sent. It does this
by indicating that no changes have been made to DNS records, which
would otherwise be large and a burden to transmit frequently.

## Technique Overview

This document proposes a new "nothing new" NN flag, a LARGE
Redirection Resource record type, and describes how these can
integrate with current and future DNSSEC DNSKEY and RRSIG records.

This document proposes two technical mechanisms for signaling that
resource records have not changed since a previously obtained set, and
thus do not need to be re-fetched.  This potentially saves significant
resources on both the client and server.  These optimizations include:

- A new Nothing New (NN) DNS bit, to be used in conjunction with the
  Truncated (TC) bit that indicates the requested records have not
  been changed recently, and thus cached data is sufficient fro use.
  See {{NN}} for details.

- A LARGE resource record ({{LARGE}}) that serves as a hint about what
  version of a record is current and whether or not a client needs to
  refetch its contents.

The trustability of these unsigned signals is discussed in
{{security}}.

The simple goal of these new features is to reduce the necessary
number of large responses from authoritative servers when
communicating with conforming resolver clients.  Effectively, these
mechanisms allow for signaling both:

1. If a recursive resolver has data in its cache, it may keep using it
   (assuming the cached DNSSEC signatures are still valid if it is
   validating).
2. A version number of the data requested to check against a
   resolver's cache, providing a hint about whether the data in a
   resolvers cache is actually old or the same.

# Conventions and Definitions {#definitions}

{::boilerplate bcp14-tagged}

# The Nothing New flag {#NN}

This document defines a Nothing New (NN) flag within the EDNS0 option
header.  This flag SHOULD be set by recursive resolvers that support
this specification. This flag SHOULD be set by authoritative servers
that support this specification and are returning a Truncation
Response (TC) bit to indicate that nothing has changed recently in the
requested resource record.  If the authoritative server is unable to
determine that nothing new has changed with respect to that resource
record, it MUST NOT set the NN bit in its response period.

In short the NN flag is a signal that can be sent along with the
Truncated Response (TC) flag to indicate both that data truncation has
occurred to comply with packet size limits {{RFC6891}} and that any
data with a still valid signature validity may be continued to use
instead of refetching the data.

This flag SHOULD be accompanied with a LARGE ({{LARGE}}) Resource
Record as well.  If the DNSSEC signature on the LARGE RR can fit
within the response, it MUST be included.  If the DNSSEC signature
on the LARGE RR cannot fit within the response, the LARGE RR SHOULD be
sent without it.

# The LARGE Resource Record {#LARGE}

The LARGE RR is a hint to the resolver about the freshness of the data
at the server compared to the freshness of the data within the
resolver's cache.

The RR contains the following fields:

- IDENTIFIER: A 16-bit serial number field that must be unique within the
  signature lifetime of the data it represents.  Resolvers can use
  this information to determine if the record they have available
  matches the value not sent by the upstream server.

The name of this record MUST match the name of the record being
requested in the query.

## Selecting serial numbers

The identifier field MUST be selected from a unique set of values that
will not duplicate during the lifetime of the DNSSEC signatures
period.  Authoritative servers which auto-generate this field can use
various forms of mechanisms, such as cryptographic hashes, incremental
serial numbers, carefully constructed timestamps, fields and values
from the data that it represents, as long as the uniqueness constraint
is properly observed.

The SOA serial number of the zone SHOULD NOT be used as LARGE record
serial numbers unless it is expected that all records in a zone are
likely to change at the same time the SOA is ever changed.  EG, highly
dynamic zones will have their SOA changing so frequently that it is
pointless to use them to indicate changes relating to otherwise fairly
static records, like DNSKEYs.

## Discussion: Alternative LARGE formats

Could be a TXT style record with the more modern key=value syntax, at
the cost of a size increase.

## Discussion: Alternative LARGE record placement

This could be done with an underbar label instead, with something like
a _large.example.com record instead.  (this is more difficult than it
sounds and probably won't work well in practice)

## Discussion: Signaling to the parent with a LARGE record

Another option is to actually have the resolver send a signal to the
parent about its cache using an embedded LARGE record within an EDNS0
packet so that the parent knows whether or not something is new.

# Use with DNSSEC {#DNSSEC}

The use of these techniques within DNSSEC is especially tricky even
while other uses may be more straight forward, as RRSIG records
themselves are frequently large but are needed to validate the data.

# Security Considerations {#security}

Obviously, using unsigned data to decide whether or not to retrieve
signed data is a security concern. Having said that, there are already
other specifications that show how to old data when the parental
server cannot be contacted ({{RFC8767}}).

This document merely provides a specification for the parental agent
to deliberately say "nothing new". If there is a machine in the middle
spoofing this signal, it already has an attack vector to cause a
resolver to use stale data by simply dropping query or response so the
resolver falls back to its stale cache.  At most, with this proposal
clients will end up using old data, which is already the case, albeit
faster than waiting for a timeout.

# IANA Considerations

TBD

--- back

# Acknowledgments
{:numbered="false"}

The bad-idea fairy contributed greatly to the ideas behind this document.

Joe Ably had constructive advice to offer, even though he may not
actually agree with the bad ideas in this document.

TBD

