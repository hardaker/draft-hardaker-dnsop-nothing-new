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

informative:
  BCP237:  # DNSSEC

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

The trustability of these unsigned signals is discussed in {{trustability}}.

# Conventions and Definitions {#definitions}

{::boilerplate bcp14-tagged}

# The Nothing New flag {#NN}

# The LARGE Resource Record {#LARGE}

# Trustability {#trustability}

# Security Considerations {#security}

# IANA Considerations

--- back

# Acknowledgments
{:numbered="false"}

TBD

