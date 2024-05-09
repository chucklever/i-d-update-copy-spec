---

title: "Updates to the Specification of the Network File System version 4.2 COPY Operation"
abbrev: "Update COPY Specification"
category: std

docname: draft-cel-nfsv4-update-copy-spec-latest
pi: [toc, sortrefs, symrefs, docmapping]
stand_alone: yes
v: 3

submissiontype: IETF
ipr: trust200902
area: "Web and Internet Transport"
workgroup: "Network File System Version 4"
obsoletes:
updates: rfc7862
keyword:
 - NFSv4.2
 - COPY
 - offload

author:
 -
    fullname: Chuck Lever
    organization: Oracle Corporation
    abbrev: Oracle
    country: United States of America
    email: chuck.lever@oracle.com

normative:
  RFC7862:

venue:
  group: nfsv4
  type: Working Group
  mail: nfsv4@ietf.org
  arch: https://mailarchive.ietf.org/arch/browse/nfsv4/
  repo: https://github.com/chucklever/i-d-update-copy-spec
  latest: https://chucklever.github.io/i-d-update-copy-spec/#go.draft-cel-nfsv4-update-copy-spec.html

--- abstract

This document updates the specification of the NFSv4.2 COPY operation.
Based on implementation experience, several clarifications to the
specification are made. These changes are not expected to alter the
on-the-wire behavior of current implementations.

--- middle

# Introduction

{{RFC7862}} introduces a mechanism to the NFSv4 protocol for NFS clients
to request that an NFS server copy data from one file to another.
Because the data copy happens on the server, it avoids the movement of file
data between client and server during the copy operation.
This reduces latency, network bandwidth requirements, and the exposure
of file data to third parties during the copy.

This document enumerates several areas where specification wording in
{{RFC7862}} could be clarified or made explicit in order to better
guarantee interoperation.
They are mostly errors of omission which allow interoperability gaps to
arise due to subtleties and ambiguities in the original specification of
the COPY operation.

# Requirements Language

{::boilerplate bcp14-tagged}

# Synchronous versus Asynchronous COPY

The NFSv4.2 protocol is designed so that an NFSv4.2 server
is considered protocol compliant whether it implements the COPY
operation or not. However, COPY comes in two distinct flavors:

  - synchronous, where the server reports the final status of the
    operation directly in the response to the client's COPY request
  - asynchronous, where the server agrees to report the status of
    the operation at a later time via a callback operation.

{{RFC7862}} does not take a position on whether a client or server
is mandated to implement either or both forms of COPY.
However, the implementation requirements for these two
forms of the COPY operation are quite distinct from each other,
and some implementers have chosen to avoid the more complex
asynchronous form of COPY.

## Detecting Support For COPY

To interoperate successfully, the client and server must be able
to determine which forms of COPY are implemented and fall back to
a normal READ/WRITE-based copy if necessary.

Given the operation of the signaling in the ca_synchronous field
as described in {{Section 15.2.3 of RFC7862}}, an implementation
that supports COPY MUST support either only synchronous copy or
both synchronous and asynchronous copy.

## Mandatory-To-Implement Operations

The synchronous form of COPY does not need a client or server to
implement the OFFLOAD_CANCEL, OFFLOAD_STATUS, or CB_OFFLOAD operations.
Moreover, COPY_NOTIFY is required only when an implementation
provides inter-server copy offload. Thus a minimum viable
synchronous-only copy implementation can get away with implementing
only the COPY operation and can leave the others unimplemented.

The asynchronous form of COPY is not possible without the
implementation of CB_OFFLOAD, and not reliable without the
implementation of OFFLOAD_STATUS. The original specification of
copy offload does not make these two operations mandatory-to-implement
when an implementation claims to support asynchronous COPY. The
addition of the following text makes this requirement clear:

> When an NFS server implementation provides an asynchronous copy
> capability, it MUST implement the CB_OFFLOAD and OFFLOAD_STATUS
> operations.

## COPY state ID Lifetime Requirements {#lifetime}

A server that implements only synchronous copy does not require
the stricter COPY state ID lifetime requirements described in
{{Section 4.8 of RFC7862}}. A state ID used with a synchronous
copy behaves lives as long as a state ID used for non-copy-related
purposes.

Regarding asynchronous COPY operations,
the second paragraph of {{Section 4.8 of RFC7862}} states:

> A copy offload stateid will be valid until either (A) the client or
> server restarts or (B) the client returns the resource by issuing an
> OFFLOAD_CANCEL operation or the client replies to a CB_OFFLOAD
> operation.

This paragraph is unclear about what "client restart" means, at least
in terms of what actions a server should take and when, how
long a COPY state ID is required to remain valid, and how a client
needs to act during state recovery. A stronger statement about
COPY state ID lifetime improves the guarantee of interoperability:

> An NFS server MUST retain a COPY state ID, except as follows.
> An NFS server MAY invalidate a COPY state ID in the following
> circumstances:
>
> o The server instance restarts.
>
> o The server expires the owning client's lease.
>
> o The server receives an EXCHANGE_ID or DESTROY_CLIENTID from the
>   owning client that results in the destruction of the client's
>   lease.
>
> o The server receives an OFFLOAD_CANCEL operation from the owning
>   client that matches the COPY state ID.
>
> o The server receives a reply to a CB_OFFLOAD operation from the
>   owning client that matches the COPY state ID.

Implementers have found the following behavior to work well for
clients when recovering state after a server restart:

> When an NFS client discovers that a server instance has restarted,
> it must recover state, including the state that manages offloaded
> copy operations. When an NFS server restart is detected, the
> client purges existing COPY state and redrives incomplete COPY
> operations from the beginning.

# Short COPY results

In cases where a COPY operation might take a long while, it is useful
to prevent server resources from blocking. A server implementation
might, for example, notice that a COPY operation is taking longer
than a few seconds and stop it.

{{Section 15.2.3 of RFC7862}} states:

> If a failure does occur for a synchronous copy, wr_count will be set
> to the number of bytes copied to the destination file before the
> error occurred.

This text considers only a failure status and not a short COPY, where
the COPY response contains a byte count shorter than the client's
request and a final status of NFS4_OK. Both the Linux and FreeBSD
implementations of the COPY operation truncate large COPY requests
in this way. The reason for returning a short COPY result is that
the NFS server has need to break up a long byte range to schedule
its resources more fairly amongst its clients.

The following text makes a short COPY result explicitly permissible:

> If a server chooses to terminate a COPY before it has completed
> copying the full requested range of bytes, either because of a
> pending shutdown request, an administrative cancel, or because
> the server wants to avoid a possible denial of service, it MAY
> return a short COPY result, where the response contains the
> actual number of bytes copied and a final status of NFS4_OK.
> In this way, a client can send a subsequent COPY for the
> remaining byte range, ensure that forward progress is made.

# OFFLOAD_STATUS

Paragraph 2 of {{Section 15.9.3 of RFC7862}} states:

> If the optional osr_complete field is present, the asynchronous
> operation has completed.  In this case, the status value indicates
> the result of the asynchronous operation.

The use of the term "optional" can be (and has been) construed to mean
that a server is not required to set that field to one, ever. This is
due to the conflation of "optional" with the common use of the compliance
keyword OPTIONAL in other NFS-related documents.

Moreover, this field (as an XDR data item) is always present. The
XDR definition does not permit the server not to include the field
in its response. The imprecision of the construct "If the optional...
field is present" can be confusing.

The following replacement makes it more clear what was originally
intended:

> To process an OFFLOAD_STATUS operation, the server must first
> find a COPY operation that matches the COPY state ID.
>
> If that COPY operation is still ongoing, the server forms a response
> with an osr_complete array containing zero elements, fills in the
> number of bytes processed by the COPY operation, and sets the status
> code for the OFFLOAD_STATUS operation to NFS4_OK.
>
> If the matching copy has completed but the server has not processed
> the client's CB_OFFLOAD reply, the server forms a response with an
> osr_complete array containing one element which is set to the final
> status code of the copy operation. It fills in the number of bytes
> that were processed by the COPY operation, and sets the status
> code for the OFFLOAD_STATUS operation to NFS4_OK.
>
> If no copy operation matches the presented COPY state ID, the
> server sets the status code for the OFFLOAD_STATUS operation to
> NFS4ERR_BAD_STATEID.

Since a single-element osr_complete array contains the status code of
a COPY operation, the specification should state explicitly that:

> When a single element is present in osr_complete, that element
> MUST contain only one of status codes permitted for the COPY
> operation (see {{Section 11.2 of RFC7862}}) or NFS4_OK.

# Asynchronous Copy Completion Reliability

Typically, NFSv4 server backchannel operations are not retransmitted.
There are common scenarios where lack of a retransmit can result in
a backchannel operation getting dropped entirely. Such scenarios
include:

  - The server dropped the connection because it lost a forechannel
    NFSv4 request and wishes to force the client to retransmit
  - The GSS sequence number window under-flowed
  - A network partition occurred

In these cases, all pending NFSv4 callback operations are commonly
lost.

NFSv4 clients and servers can recover when operations such as
CB_RECALL and CB_GETATTR go missing. After a delay, the server
revokes the delegation and operation continues.

A lost CB_OFFLOAD means that a client workload is waiting for a
completion event that never arrives, unless that client has a
mechanism for probing the pending COPY (usually this means
sending an OFFLOAD_STATUS). The specification can make this
implementation quality issue more clear:

> NFSv4 servers are not required to retransmit lost backchannel
> operations. If an NFS client implements an asynchronous copy
> capability, it MUST implement a mechanism for recovering from
> a lost CB_OFFLOAD operation. The NFSv4.2 protocol provides
> one such mechanism in the form of OFFLOAD_STATUS.

# Security Considerations

One important responsibility of an NFS server implementation is
to manage a finite set of resources in a way that minimizes the
opportunity for NFS clients to maliciously or unintentionally
cause a denial-of-service scenario.

## Limiting size of individual COPY operations

Server implementations have so far chosen to limit the byte range
of COPY operations, either statically (by setting a fixed limit
on the number of bytes a single COPY can process, where the server
truncates the copied byte range) or dynamically (by setting a
timeout). In either case, the server returns a short COPY result.

Client implementations accommodate this limit by sending a
fresh COPY for the remainder of the byte range, until the
full byte range has been processed.

## Limiting the number of outstanding asynchronous COPY operations

It is easily possible for NFS clients to request more asynchronous
COPY operations than server resources can handle. For example, a
client could create a large file, and then request multiple copies
of that file's contents to /dev/null.

A good quality server implementation SHOULD block clients from
starting many COPY operations. The implementation can apply a
fixed per-client limit, a per-server limit, or a dynamic limit
based on available resources. When that limit has been reached,
subsequent COPY requests will receive NFS4ERR_OFFLOAD_NO_REQS
in response until more server resources become available.

## Pruning COPY state on server

A related issue is how much COPY state can accrue on a server
when CB_OFFLOAD requests are lost. The requirements in {{lifetime}}
require a server to retain this abandoned COPY state indefinitely.

A good quality server implementation SHOULD launder the list of
abandoned COPY state on occasion so that it does not grow without
bounds.

# IANA Considerations

This document requests no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

Special thanks to Rick Macklem, Olga Kornievskaia, and Dai Ngo for
their insights and work on implementations of NFSv4.2 COPY.

The author is grateful to Bill Baker, Greg Marsden, and Martin Thomson
for their input and support.

Special thanks to Transport Area Directors Martin Duke and
Zaheduzzaman Sarker, NFSV4 Working Group Chairs Chris Inacio and
Brian Pawlowski, and NFSV4 Working Group Secretary Thomas Haynes for
their guidance and oversight.
