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

The DNS protocol has increasingly needed to carry larger records than
it was originally designed to carry.  This has resulted in performance
impacts due to both the size increases and requiring TCP instead of
only UDP.  Of particular note is the expected large increase in
records relating to post-quantum signing algorithms.  To help
mitigate, but not entirely prevent, these impacts, this document
proposes a new "nothing new" NN flag, a LARGE Redirection Resource
record type, and how these can integrate with current and future
DNSSEC DNSKEY and RRSIG records.

Recent research into the impacts of Post Quantum Cryptography (PQC)
sized keys being introduce into DNSSEC has shown that every
potential algorithm poses significant impacts into signing,
verification and potential packet sizes.  The resulting expectation is
that a significant increase in requiring TCP for both DNSKEY record
retrieval and even any record retrieval, as the signature sizes alone
will be significantly larger than easily distributable over UDP.

This document proposes a number of new mechanisms for signaling that
resource records have not changed, and thus do not need to be
refetched, potentially saving significant resources on both the client
and server.  These optimizations include:

- A new Nothing New (NN) DNS flag that indicates the requested records
  have not been changed recently, and thus cached data is sufficient
  fro use.  See {{NN}} for details.

- A LARGE resource record that both can serve as a hint about what
  version of a record is current and how resolvers can query for the
  entire record using either TCP or multiple UDP requests {{LARGE}}.

The trustability of these unsigned signals is discussed in
{{security}}.

The goal of these new features is to reduce the number of large
responses necessary when communicating with conforming resolver
clients.  Effectively, these mechanisms allow for signaling both:

1. If you have data in your cache, you may keep using it, assuming the
   cached DNSSEC signatures are still valid.
2. A serial number of the data requested to check against your cache,
   in case it actually has changed.

# Conventions and Definitions {#definitions}

{::boilerplate bcp14-tagged}

# The Nothing New flag {#NN}

In short, the NN flag in the DNS Header Flags is a signal that can be
sent along with the Truncated Response (TC) flag to indicate both that
data truncation has occurred to comply with packet size limits
{{RFC6891}} and that any data with a still valid signature validity
may be continued to use instead of refetching the data.

This flag SHOULD be accompanied with a LARGE ({{LARGE}}) Resource
Record as well.  If the DNSSEC signature on the LARGE RR can fit
within the response, it MUST be included.  If the DNSSEC signature
on the LARGE RR cannot fit within the response, the LARGE RR SHOULD be
sent without it.

# The LARGE Resource Record {#LARGE}

The LARGE RR is a hint to the resolver about how to fetch the larger
record.  It includes mechanisms for where and how to fetch the data
over both UDP and TCP and hints about whether fetching the data is
even necessary.

The RR contains the following fields:

- IDENTIFIER: A 16-bit serial number field that must be unique within the
  signature lifetime of the data it represents.  Resolvers can use
  this information to determine if the record they have available
  matches the value not sent by the upstream server.  (more details
  later)

- TCP_TYPE: The 16-bit Resource Record type that can used for fetching the full
  contents over TCP.  Generally this will be the same Resource Record
  type that the LARGE record is covering, with some rare exceptions.

- UDP_TYPE: The 16-bit starting Resource Record type for fetching parts of the larger
  record by requesting individual pieces of the record over UDP
  instead.  A UDP_TYPE field of zero indicates there is no RR type where multiple segments
  can be downloaded.

- UDP_COUNT: An 8-bit field specifying the number of UDP Resource
  Records, starting with the UDP_TYPE RR, to fetch and concatinated
  when retrieving the larger value.

## Selecting serial numbers

The SOA serial number SHOULD NOT be used as LARGE record serial
numbers unless it is expected that all records in a zone are likely to
change at the same time the SOA is ever changed.  EG, highly dynamic
zones will have their SOA changing so frequently that it is pointless
to use them to indicate changes relating to otherwise fairly static
records, like DNSKEYs.

## Alternative LARGE format

Could be a TXT style record with the more modern key=value syntax, at
the cost of a size increase.

## Alternative LARGE record placement

This could be done with an underbar label instead, with something like
a _large.example.com record instead.

## The UDP support may or may not be desired

The UDP support is sort of a hack, but could be useful.  Or dropped.

# Use with DNSSEC {#DNSSEC}

The use of these techniques within DNSSEC is especially tricky even
while other uses may be more straight forward, as RRSIG records
themselves are frequently large but are needed to validate the data.

# Security Considerations {#security}

Obviously, using unsigned data to decide whether or not to retrieve
signed data is a security concern. Having said that, there are already
other specifications that show how to use unsigned data when the
parental server cannot be contacted ({{RFC8767}}).

This document merely provides a specification for the parental agent
to deliberately say "nothing new". And thus, if there is a machine in
the middle spoofing this signal, there is already a machine in the
middle that can cause a child to use stale data by simply dropping packets.
At most, clients will end up using old data, which is already the case.

# IANA Considerations

TBD

--- back

# Acknowledgments
{:numbered="false"}

The bad-idea fairy contributed greatly to the ideas behind this document.

TBD

