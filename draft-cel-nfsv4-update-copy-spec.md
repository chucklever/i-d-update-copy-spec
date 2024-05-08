---
title: "Clarifications to the Specification of the Network File System version 4.2 COPY Operation"
abbrev: "Update COPY Specification"
category: std

docname: draft-cel-nfsv4-update-copy-spec-latest
submissiontype: IETF
ipr: trust200902
v: 3
area: Transport
workgroup:  Network File System Version 4
obsoletes:
updates: rfc7862
keyword:
 - Network File System version 4
 - COPY offload

author:
 -
    fullname: Chuck Lever
    organization: Oracle Corporation
    abbrev: Oracle
    country: United States of America
    email: chuck.lever@oracle.com

normative:
RFC7862:

informative:

--- abstract

This document updates the specification of the NFSv4.2 COPY operation.
Based on implementation experience, several clarifications to the
specification are made.

--- note_Note

This note is to be removed before publishing as an RFC.

Discussion of this draft occurs on the [NFSv4 working group mailing
list](nfsv4@ietf.org), archived at
[](https://mailarchive.ietf.org/arch/browse/nfsv4/). Working Group
information is available at
[](https://datatracker.ietf.org/wg/nfsv4/about/).

Submit suggestions and changes as pull requests at
[](https://github.com/chucklever/i-d-update-copy-spec). Instructions
are on that page.

--- middle

# Introduction

{{RFC7862}} introduces a mechanism to the NFSv4 protocol for NFS clients
to request that an NFS server copy data from one file to another.
The data copy happens on the server, avoiding the movement of file
data on the network between client and server during the copy operation.
This reduces latency, network bandwidth requirements, and the exposure
of file data to third parties during the copy.

As a result of implementing the COPY operation as described by {{RFC7862}},
the author found several areas where specification wording could be
clarified or made explicit in order to better guarantee interoperation.

This document enumerates these issues. They are mostly errors of
omission which allow interoperability gaps to arise due to subtleties
and ambiguities in the original specification of the COPY operation.

# Requirements Language

{::boilerplate bcp14-tagged}

[ cel: distinguish between the use of compliance keywords in
quotations versus in the document's body text. ]

# Synchronous versus Asynchronous COPY

The NFSv4.2 protocol is designed so that an NFSv4.2 server
is considered protocol compliant whether it implements the COPY
operation or not. However, COPY comes in two distinct flavors:
- synchronous, where the server reports the final status of the
  operation directly in the response to the client's COPY request
- asynchronous, where the server agrees to report the status of
  the operation at a later time via a callback operation.

The requirements for these two forms of the COPY operation are
quite distinct from each other.

## Mandatory-To-Implement Operations

{{RFC7862}} does not take a position on whether a client or server
is REQUIRED to implement both forms of COPY if it claims to support
copy offload at all.
To interoperate successfully, the client and server must be able
to determine what forms of COPY are implemented and fall back to
a normal READ/WRITE-based copy if necessary.
To implement only synchronous COPY, the client simply always asserts
ca_synchronous. The server is free to return a status of
NFS4ERR_OFFLOAD_NO_REQS if it cannot provide a synchronous COPY.
[ cel: more interop analysis to follow ]

The synchronous form of COPY does not require a client or server to
implement the OFFLOAD_CANCEL, OFFLOAD_STATUS, or CB_OFFLOAD operations.

Note that COPY_NOTIFY is required only when an implementation
provides inter-server copy offload.

[ cel: maybe an MTI table would help? ]

The asynchronous form of COPY is not possible without the
implementation of CB_OFFLOAD, and not reliable without the
implementation of OFFLOAD_STATUS. The spec does not make these
two operations mandatory-to-implement when an implementation
claims to support asynchronous COPY.

Notes:
* When is OFFLOAD_STATUS mandatory to implement, if ever?
* When is CB_OFFLOAD mandatory to implement, if ever?

## COPY state ID Lifetime Requirements

A server that implements only synchronous COPY does not require
the strict COPY state ID lifetime requirements described in
{{Section 4.8 of RFC7862}}. This is another way that synchronous
COPY is simpler to implement.

* What is the COPY state ID lifetime for a synchronous COPY? An asynchronous COPY?

Regarding asynchronous COPY operations,
the second paragraph of {{Section of 4.8 of RFC7862}} states:

   A copy offload stateid will be valid until either (A) the client or
   server restarts or (B) the client returns the resource by issuing an
   OFFLOAD_CANCEL operation or the client replies to a CB_OFFLOAD
   operation.

This paragraph is unclear about what "client restart" means, at least
in terms to what actions a server should take and when, and thus how
long a COPY state ID is required to remain valid.

[ cel: go into client state recovery best practices ]

## Short COPY results

In cases where a COPY operation might take a long while, it is useful
to prevent server resources from blocking. A server implementation
might, for example, notice that a COPY operation is taking longer
than a few seconds and stop it.

{{ Section 15.2.3 of RFC 7862 }} states:

   If a failure does occur for a synchronous copy, wr_count will be set
   to the number of bytes copied to the destination file before the
   error occurred.

This text considers only a failure status and not a short COPY, where
the COPY response contains a byte count shorter than the client's
request, but a final status of NFS4_OK. Both the Linux and FreeBSD
implementations of the COPY operation truncate large COPY requests
in this way. The specification could add the following:

   If a server chooses to terminate a COPY before it has completed
   copying the full requested range of bytes, either because of a
   pending shutdown request, an administrative cancel, or because
   the server wants to avoid a possible denial of service, it MAY
   return a short COPY result, where the response contains the
   actual number of bytes copied and a final status of NFS4_OK.
   In this way, a client can send a subsequent COPY for the
   remaining byte range, ensure that forward progress is made.


## OFFLOAD_STATUS

Paragraph 2 of {{ Section 15.9.3 of RFC 7862}} states:

   If the optional osr_complete field is present, the asynchronous
   operation has completed.  In this case, the status value indicates
   the result of the asynchronous operation.

The use of the term "optional" can be (and has been) construed to mean
that a server is not required to set that field to one, ever. This is
due to the conflation of "optional" with the compliance keyword
OPTIONAL.

Moreover, this field (as an XDR data item) is always present. The
XDR definition does not permit the server not to include the field
in its response.

Perhaps the following replacement makes it more clear what was
originally intended:

   To process an OFFLOAD_STATUS operation, the server must first
   find a COPY operation that matches the COPY state ID.

   If that COPY operation is still ongoing, the server forms a response
   with an osr_complete array containing zero elements, fills in the
   number of bytes processed by the COPY operation, and sets the status
   code for the OFFLOAD_STATUS operation to NFS4_OK.

   If the matching copy has completed, but the server has not processed
   the client's CB_OFFLOAD reply, the server forms a response with an
   osr_complete array containing one element, which is set to the final
   status code of the copy operation. It also fills in the number of
   bytes that were processed by the COPY operation, and sets the status
   code for the OFFLOAD_STATUS operation to NFS4_OK.

   If no copy operation matches the presented COPY state ID, the
   server sets the status code for the OFFLOAD_STATUS operation to
   NFS4ERR_BAD_STATEID.

Since a single-element osr_complete array contains the status code of
a COPY operation, it bears stating explicitly that:

   When a single element is present in osr_complete, that element
   MUST contain one of status codes permitted for the COPY operation
   (see {{ Section 11.2 of RFC 7862 }}), or NFS4_OK.

## Completion Reliability

* Is a server required to retransmit CB_OFFLOAD?
* Is a server permitted to return a short COPY result?

# Security Considerations

* Pruning COPY state on server
* Limiting size of individual COPY operations


# IANA Considerations

RFC Editor: In the following subsections, please replace RFC-TBD with
the RFC number assigned to this document. Furthermore, please remove
this Editor's Note before this document is published.

This document requests no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

The author is grateful to Bill Baker, Greg Marsden, and Martin Thomson
for their input and support.

Special thanks to Transport Area Directors Martin Duke and
Zaheduzzaman Sarker, NFSV4 Working Group Chairs Chris Inacio and
Brian Pawlowski, and NFSV4 Working Group Secretary Thomas Haynes for
their guidance and oversight.
