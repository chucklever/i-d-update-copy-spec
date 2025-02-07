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
  RFC5661:
  RFC7862:
  RFC7863:
  RFC8881:

informative:
  XCOPY:
    -: XCOPY
    target: https://www.t10.org/ftp/t10/document.99/99-143r1.pdf
    title: "T10/99-143r1: 7.1 EXTENDED COPY command"
    author:
      -
        ins: Unknown
        name: Author Unknown
        org: T10 Organization
    seriesinfo:
      ISBN:
      DOI:
    date: 1999-04-02
    format:
      PDF: https://www.t10.org/ftp/t10/document.99/99-143r1.pdf

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

{{RFC7862}} introduces a facility to the NFSv4 protocol for NFS clients
to request that an NFS server copy data from one file to another.
Because the data copy happens on the NFS server, it avoids the transit
of file data between client and server during the copy operation.
This reduces latency, network bandwidth requirements, and the exposure
of file data to third parties to handle the copy request.

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
forms of copy offload are quite distinct from each other,
and some implementers have chosen to avoid the more complex
asynchronous form of COPY.

## Detecting Support For COPY

{{Section 4.1.2 of RFC7862}} states:

> Inter-server copy, intra-server copy, and intra-server clone are each
> OPTIONAL features in the context of server-side copy.  A server may
> choose independently to implement any of them.  A server implementing
> any of these features may be REQUIRED to implement certain
> operations.  Other operations are OPTIONAL in the context of a
> particular feature (see Table 5 in Section 13) but may become
> REQUIRED, depending on server behavior.  Clients need to use these
> operations to successfully copy a file.

The existing specification distinguishes between implementations that
support inter-server or intra-server copy, but does not differentiate
between implementations that support synchronous versus asynchronous
copy.




To interoperate successfully, the client and server must be able
to determine which forms of COPY are implemented and fall back to
a normal READ/WRITE-based copy if necessary. The following
addition makes this more clear:

> Given the operation of the signaling in the ca_synchronous field
> as described in {{Section 15.2.3 of RFC7862}}, an implementation
> that supports COPY MUST support either only synchronous copy or
> both synchronous and asynchronous copy.

## Mandatory-To-Implement Operations

The synchronous form of copy offload does not need a client or
server to implement the OFFLOAD_CANCEL, OFFLOAD_STATUS, or
CB_OFFLOAD operations.
Moreover, the COPY_NOTIFY operation is required only when an
implementation provides inter-server copy offload. Thus a minimum
viable synchronous-only copy implementation can get away with
implementing only the COPY operation and can leave the others
unimplemented.

The asynchronous form of copy offload is not possible without
the implementation of CB_OFFLOAD, and not reliable without the
implementation of OFFLOAD_STATUS. The original specification of
copy offload does not make these two operations mandatory-to-implement
when an implementation claims to support asynchronous COPY. The
addition of the following text makes this requirement clear:

> When an NFS server implementation provides an asynchronous copy
> capability, it MUST implement the CB_OFFLOAD and OFFLOAD_STATUS
> operations.

## COPY state ID Lifetime Requirements {#lifetime}

An NFS server that implements only synchronous copy does not require
the stricter COPY state ID lifetime requirements described in
{{Section 4.8 of RFC7862}}. A state ID used with a synchronous
copy lives until the COPY operation has completed.

Regarding asynchronous copy offload,
the second paragraph of {{Section 4.8 of RFC7862}} states:

> A copy offload stateid will be valid until either (A) the client or
> server restarts or (B) the client returns the resource by issuing an
> OFFLOAD_CANCEL operation or the client replies to a CB_OFFLOAD
> operation.

This paragraph is unclear about what "client restart" means, at least
in terms of what specific actions a server should take and when, how
long a COPY state ID is required to remain valid, and how a client
needs to act during state recovery. A stronger statement about
COPY state ID lifetime improves the guarantee of interoperability:

> When a COPY state ID is used for an asynchronous copy, an NFS
> server MUST retain the COPY state ID, except as follows below.
> An NFS server MAY invalidate and purge a COPY state ID in the
> following circumstances:
>
> o The server instance restarts.
>
> o The server expires the owning client's lease.
>
> o The server receives an EXCHANGE_ID or DESTROY_CLIENTID request
>   from the owning client that results in the destruction of that
>   client's lease.
>
> o The server receives an OFFLOAD_CANCEL request from the owning
>   client that matches the COPY state ID.
>
> o The server receives a reply to a CB_OFFLOAD request from the
>   owning client that matches the COPY state ID.

Implementers have found the following behavior to work well for
clients when recovering state after a server restart:

> When an NFSv4 client discovers that a server instance has restarted,
> it must recover state associated with files on that server, including
> state that manages offloaded copy operations. When an NFS server
> restart is detected, the client purges existing COPY state and
> redrives its incompleted COPY requests from their beginning. No
> other recovery is needed for pending asynchronous copy operations.

# Status Codes, Their Meanings, and Their Usage

## Status Codes for the CB_OFFLOAD Operation

{{Section 16.1.3 of RFC7862}} describes the CB_OFFLOAD command, but
provides no information, normative or otherwise, about the NFS client's
callback service is to use CB_OFFLOAD's response status codes. The set
of permitted status codes is listed in {{Section 11.3 of RFC7862}}.

The usual collection of status codes related to compound structure
and session parameters are available.
However, Section 11.3 also lists NFS4ERR_BADHANDLE, NFS4ERR_BAD_STATEID,
and NFS4ERR_DELAY.

{{Section 15.1.2.1 of RFC8881}} defines NFS4ERR_BADHANDLE this way:

> Illegal NFS filehandle for the current server. The current filehandle
> failed internal consistency checks.

There is no filesystem on an NFS client to determine whether a filehandle
is valid. What other checking should the client's callback server perform?

Instead of NFS4ERR_BADHANDLE, the NFS client's callback service might
instead return either NFS4ERR_INVAL (which is currently not a permitted
status code for CB_OFFLOAD) or NFS4ERR_SERVERFAULT.

{{Section 15.1.5.2 of RFC8881}} states that NFS4ERR_BAD_STATEID means that

> A stateid does not properly designate any valid state.

But note that:

 * The NFS server manages the state ID that appears in the wr_callback_id
   field of the CB_OFFLOAD response. The NFS client's callback service is
   not in a position to validate that state ID since it did not create it.

 * The client might receive and process the CB_OFFLOAD before it has
   received and processed the COPY that contains the copy state ID it has
   to match the CB_OFFLOAD against. In that case, the copy state ID is
   certainly valid but the client has not yet been made aware of it.

 * The NFS client's callback service will already have validated the
   client ID contained in the CB_SEQUENCE operation that accompanies
   the CB_OFFLOAD. State ID sequence number validation does not seem
   relevant for copy state IDs.

 * Upon receipt of NFS4ERR_BAD_STATEID, the receiver typically initiates
   state recovery. In this case, there is no protocol defined anywhere
   that can recover the state ID.

One must conclude that implementations of CB_OFFLOAD MUST NOT use
NFS4ERR_BAD_STATEID. We recommend that it be removed from the list of
permissible state codes for CB_OFFLOAD.

{{Section 15.1.1.3 of RFC8881}} has this to say about NFS4ERR_DELAY:

> For any of a number of reasons, the replier could not process this
> operation in what was deemed a reasonable time. The client should wait
> and then try the request with a new slot and sequence value.

When the NFS client does not recognize the copy state ID in the
wr_callback_id field, it's probable that the matching COPY response
has not yet been processed on the client. A simple and appropriate
response to that situation is for the NFS client's callback service
to respond with NFS4ERR_DELAY.

A refreshed version of Table 3 in {{RFC7862}} might look like this:

| Callback Operation | Errors |
| CB_OFFLOAD | NFS4ERR_BADXDR, NFS4ERR_DELAY, NFS4ERR_OP_NOT_IN_SESSION, NFS4ERR_REP_TOO_BIG, NFS4ERR_REP_TOO_BIG_TO_CACHE, NFS4ERR_REQ_TOO_BIG, NFS4ERR_RETRY_UNCACHED_REP, NFS4ERR_SERVERFAULT, NFS4ERR_TOO_MANY_OPS |

Valid Error Returns for Each New Protocol Callback Operation

## Status Codes for the OFFLOAD_CANCEL and OFFLOAD_STATUS Operations

The NFSv4.2 OFFLOAD_STATUS and OFFLOAD_CANCEL operations both list
NFS4ERR_COMPLETE_ALREADY as a permitted status code. However, it
is not otherwise mentioned or defined in {{RFC7862}}. {{RFC7863}}
defines a value of 10054 for that status code, but is not otherwise
forthcoming about what its purpose is.

We find a definition of NFS4ERR_COMPLETE_ALREADY in {{RFC5661}}.
The definition is directly related to the new-to-NFSv4.1
RECLAIM_COMPLETE operation, but is otherwise not used by other
operations.

We recommend removing NFS4ERR_COMPLETE_ALREADY from the list of
permissible status codes for the OFFLOAD_CANCEL and OFFLOAD_STATUS
operations.

## Status Codes Returned for Completed Asynchronous Copy Operations

Once an asynchronous copy operation is complete,
the NFSv4.2 OFFLOAD_STATUS response and the NFSv4.2 CB_OFFLOAD request
can both report a status code that reflects the success or failure of
the copy. This status code is reported in osr_complete field of the
OFFLOAD_STATUS response, and the coa_status field of the CB_OFFLOAD
request.

Both fields have a type of nfsstat4. Typically an NFSv4 protocol
specification will constrain the values that are permitted in a
field that contains an operation status code, but {{RFC7862}} does
not appear to do so. Implementers might assume that the list of
permitted values in these two fields is the same as the COPY
operation itself; that is:

>  +----------------+--------------------------------------------------+
>  | COPY           | NFS4ERR_ACCESS, NFS4ERR_ADMIN_REVOKED,           |
>  |                | NFS4ERR_BADXDR, NFS4ERR_BAD_STATEID,             |
>  |                | NFS4ERR_DEADSESSION, NFS4ERR_DELAY,              |
>  |                | NFS4ERR_DELEG_REVOKED, NFS4ERR_DQUOT,            |
>  |                | NFS4ERR_EXPIRED, NFS4ERR_FBIG,                   |
>  |                | NFS4ERR_FHEXPIRED, NFS4ERR_GRACE, NFS4ERR_INVAL, |
>  |                | NFS4ERR_IO, NFS4ERR_ISDIR, NFS4ERR_LOCKED,       |
>  |                | NFS4ERR_MOVED, NFS4ERR_NOFILEHANDLE,             |
>  |                | NFS4ERR_NOSPC, NFS4ERR_OFFLOAD_DENIED,           |
>  |                | NFS4ERR_OLD_STATEID, NFS4ERR_OPENMODE,           |
>  |                | NFS4ERR_OP_NOT_IN_SESSION,                       |
>  |                | NFS4ERR_PARTNER_NO_AUTH,                         |
>  |                | NFS4ERR_PARTNER_NOTSUPP, NFS4ERR_PNFS_IO_HOLE,   |
>  |                | NFS4ERR_PNFS_NO_LAYOUT, NFS4ERR_REP_TOO_BIG,     |
>  |                | NFS4ERR_REP_TOO_BIG_TO_CACHE,                    |
>  |                | NFS4ERR_REQ_TOO_BIG, NFS4ERR_RETRY_UNCACHED_REP, |
>  |                | NFS4ERR_ROFS, NFS4ERR_SERVERFAULT,               |
>  |                | NFS4ERR_STALE, NFS4ERR_SYMLINK,                  |
>  |                | NFS4ERR_TOO_MANY_OPS, NFS4ERR_WRONG_TYPE         |
>  +----------------+--------------------------------------------------+

However, a number of these do not make sense outside the context of
a forward channel NFSv4 COMPOUND operation, including:

NFS4ERR_BADXDR,
NFS4ERR_DEADSESSION,
NFS4ERR_OP_NOT_IN_SESSION,
NFS4ERR_REP_TOO_BIG,
NFS4ERR_REP_TOO_BIG_TO_CACHE,
NFS4ERR_REQ_TOO_BIG,
NFS4ERR_RETRY_UNCACHED_REP,
NFS4ERR_TOO_MANY_OPS

Some are temporary conditions that can be retried by the NFS server
and therefore do not make sense to report as a copy completion status:

NFS4ERR_DELAY,
NFS4ERR_GRACE,
NFS4ERR_EXPIRED

Some report an invalid argument or file object type, or some other
operational set up issue that should be reported only via the status
code of the COPY operation:

NFS4ERR_BAD_STATEID,
NFS4ERR_DELEG_REVOKED,
NFS4ERR_INVAL,
NFS4ERR_ISDIR,
NFS4ERR_FBIG,
NFS4ERR_LOCKED,
NFS4ERR_MOVED,
NFS4ERR_NOFILEHANDLE,
NFS4ERR_OFFLOAD_DENIED,
NFS4ERR_OLD_STATEID,
NFS4ERR_OPENMODE,
NFS4ERR_PARTNER_NO_AUTH,
NFS4ERR_PARTNER_NOTSUPP,
NFS4ERR_ROFS,
NFS4ERR_SYMLINK,
NFS4ERR_WRONG_TYPE

This leaves only a few sensible status codes remaining to report
issues that could have arisen during the offloaded copy:

NFS4ERR_DQUOT,
NFS4ERR_FHEXPIRED,
NFS4ERR_IO,
NFS4ERR_NOSPC,
NFS4ERR_PNFS_IO_HOLE,
NFS4ERR_PNFS_NO_LAYOUT,
NFS4ERR_SERVERFAULT,
NFS4ERR_STALE

We recommend including a section and table that gives the valid
status codes that the osr_complete and coa_status fields may contain.
The status code NFS4_OK (indicating no error occurred during the
copy operation) is not listed but should be understood to be a
valid value for these fields. The meaning for each of these values
is defined in {{Section 15.3 of RFC5661}}.

It would also be helpful to implementers to provide guidance about
when these values are appropriate to use, or when they MUST NOT be
used.

# OFFLOAD_CANCEL Implementation Notes

The NFSv4.2 OFFLOAD_CANCEL operation, described in
{{Section 15.8.3 of RFC7862}}, is used to terminate an offloaded
copy operation before it completes normally. A CB_OFFLOAD is
not necessary when an offloaded operation completes because of
a cancelation due to CB_OFFLOAD.

However, the server MUST send a CB_OFFLOAD operation if the
offloaded copy operation completes because of an administrator
action that terminates the copy early.

In both cases, a subsequent OFFLOAD_STATUS returns the number
of bytes actually copied and a status code of NFS4_OK to
signify that the copy operation is no longer running. The
server should obey the usual lifetime rules for the copy
state ID associated with a canceled asynchronous copy
operation so that an NFS client can determine the status of
the operation as usual.

The following is a recommended addendum to {{RFC7862}}:

> When an offloaded copy operation completes, the NFS server sends
> a CB_OFFLOAD callback to the requesting client. When an NFS
> client cancels an asynchronous copy operation, however, the
> server need not send a CB_OFFLOAD notification for the canceled
> copy. A client should not depend on receiving completion notification
> after it cancels an offloaded copy operation using the OFFLOAD_CANCEL
> operation.
>
> An NFS server is still REQUIRED to send a CB_OFFLOAD notification
> if an asynchronous copy operation is terminated by administrator
> action.
>
> For a canceled asynchronous copy operation, CB_OFFLOAD and
> OFFLOAD_STATUS report the number of bytes copied and a
> completion status of NFS4_OK.

# Short COPY results

When a COPY request might take a long time, an NFS server must
ensure it can continue to remain responsive to other requests.
To prevent other requests from blocking, an NFS server
implementation might, for example, notice that a COPY operation
is taking longer than a few seconds and stop it.

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

# OFFLOAD_STATUS Implementation Notes

Paragraph 2 of {{Section 15.9.3 of RFC7862}} states:

> If the optional osr_complete field is present, the asynchronous
> operation has completed.  In this case, the status value indicates
> the result of the asynchronous operation.

The use of the term "optional" can be (and has been) construed to mean
that a server is not required to set that field to one, ever. This is
due to the conflation of the term "optional" with the common use of
the compliance keyword OPTIONAL in other NFS-related documents.

Moreover, this XDR data item is always present. The protocol's XDR
definition does not permit an NFS server not to include the field
in its response.

The following text makes it more clear what was originally intended:

> To process an OFFLOAD_STATUS request, an NFS server must first
> find an outstanding COPY operation that matches the request's
> COPY state ID argument.
>
> If that COPY operation is still ongoing, the server forms a response
> with an osr_complete array containing zero elements, fills in the
> osr_count field with the number of bytes processed by the COPY
> operation so far, and sets the osr_status field to NFS4_OK.
>
> If the matching copy has completed but the server has not yet seen
> or processed the client's CB_OFFLOAD reply, the server forms a
> response with an osr_complete array containing one element which is
> set to the final status code of the copy operation. It fills in the
> osr_count field with the number of bytes that were processed by the
> COPY operation, and sets the osr_status to NFS4_OK.
>
> If the server can find no copy operation that matches the presented
> COPY state ID, the server sets the osr_status field to
> NFS4ERR_BAD_STATEID.

Since a single-element osr_complete array contains the status code of
a COPY operation, the specification needs to state explicitly that:

> When a single element is present in the osr_complete array, that
> element MUST contain only one of status codes permitted for the
> COPY operation (see {{Section 11.2 of RFC7862}}) or NFS4_OK.

# Asynchronous Copy Completion Reliability

Typically, NFSv4 server backchannel requests are not retransmitted.
There are common scenarios where lack of a retransmit can result in
a backchannel request getting dropped entirely. Common scenarios
include:

- The server dropped the connection because it lost a forechannel
  NFSv4 request and wishes to force the client to retransmit all
  of its pending forechannel requests
- The GSS sequence number window under-flowed
- A network partition occurred

In these cases, pending NFSv4 callback requests are lost.

NFSv4 clients and servers can recover when operations such as
CB_RECALL and CB_GETATTR go missing: After a delay, the server
revokes the delegation and operation continues.

A lost CB_OFFLOAD means that the client workload waits for a
completion event that never arrives, unless that client has a
mechanism for probing the pending COPY.

Typically, polling for completion means the client sends
an OFFLOAD_STATUS request. Note however that Table 5 in
{{Section 13 of RFC7862}} labels OFFLOAD_STATUS OPTIONAL.

Implementers of the SCSI protocol have reported that it is
in fact not possible to make SCSI XCOPY {{XCOPY}} reliable
without the use of polling. The NFSv4.2 COPY use case seems
no different.

The following is a recommended addendum to {{RFC7862}}:

> NFSv4 servers are not required to retransmit lost backchannel
> requests. If an NFS client implements an asynchronous copy
> capability, it MUST implement a mechanism for recovering from
> a lost CB_OFFLOAD request. The NFSv4.2 protocol provides
> one such mechanism in the form of the OFFLOAD_STATUS operation.

In addition, Table 5 should be updated to make OFFLOAD_STATUS
REQUIRED (i.e., column 3 of the OFFLOAD_STATUS row should
read the same as column 3 of the CB_OFFLOAD row in Table 6).

# Handling NFS Server Shutdown

## Graceful Shutdown

This section discusses what happens to ongoing asynchronous copy
operations when an NFS server shuts down due to an administrator
action.

When an NFS server shuts down, it typically stops accepting work
from the network. Asynchronous copy is work the NFS server has
already accepted. Normal network corking will not terminate ongoing
work; it will only stop new work from being accepted.

Thus, as an early part of NFS server shut down processing, the NFS
server must explicitly terminate ongoing asynchronous copy operations.
This includes sending CB_OFFLOAD notifications for each terminated
copy operation prior to the backchannel closing down.

To prevent the destruction of the backchannel while asynchronous
copy operations are ongoing, DESTROY_CLIENTID MUST return a status
of NFS4ERR_CLIENTID_BUSY until pending asynchronous copy operations
have terminated (see {{Section 18.50.3 of RFC8881}}).

{aside: maybe DESTROY_SESSION should be made to wait as well}

Once copy activity has completed, shut down processing can also
proceed to remove all copy completion state (copy state IDs and
copy completion status codes).

## Client Recovery Actions

In order to ensure the proper completion of asynchronous COPY
operations that were active during an NFS server restart, clients
need to track these operations and restart them as part of NFSv4
state recovery.

# Security Considerations

One critical responsibility of an NFS server implementation is
to manage its finite set of resources in a way that minimizes the
opportunity for network actors (such as NFS clients) to maliciously
or unintentionally cause a denial-of-service scenario. The following
text is an addendum to {{Section 4.9 of RFC7862}}.

> Limiting Size of Individual COPY Operations
>
> NFS server implementations have so far chosen to limit the byte
> range of COPY operations, either by setting a fixed limit on the
> number of bytes a single COPY can process, where the server
> truncates the copied byte range, or by setting a timeout). In
> either case, the NFS server returns a short COPY result.
>
> Client implementations accommodate a short COPY result by sending
> a fresh COPY for the remainder of the byte range, until the
> full byte range has been processed.
>
> An alternative approach is to convert all large synchronous copy
> requests into asynchronous copy requests, if the server supports
> asynchronous copy.
>
> Limiting the Number of Outstanding Asynchronous COPY Operations
>
> It is easily possible for NFS clients to send more asynchronous
> COPY requests than NFS server resources can handle. For example, a
> client could create a large file, and then request multiple copies
> of that file's contents to /dev/null.
>
> A good quality server implementation SHOULD block clients from
> starting many COPY operations. The implementation might apply a
> fixed per-client limit, a per-server limit, or a dynamic limit
> based on available resources. When that limit has been reached,
> subsequent COPY requests will receive NFS4ERR_OFFLOAD_NO_REQS
> in response until more server resources become available.
>
> Managing Abandoned COPY State on the Server
>
> A related issue is how much COPY state can accrue on a server
> due to lost CB_OFFLOAD requests. The mandates in {{lifetime}}
> require a server to retain abandoned COPY state indefinitely.
> A server can reject new asynchronous COPY requests using
> NFS4ERR_OFFLOAD_NO_REQS when there are many abandoned COPY
> state IDs.
>
> Considerations For The NFSv4 Callback Service
>
> There is a network service running on each NFSv4.2 client to
> handle CB_OFFLOAD operations. This service might handle only
> reverse-direction operations on an existing forward channel
> RPC transport, or it could also be available via separate
> transport connections from the NFS server.
>
> A client implementation cannot recognize the server-generated
> copy state ID contained in a CB_OFFLOAD request until it has
> received and processed the COPY response that reports that an
> asynchronous copy operation has been started and that provides
> the copy state ID to wait for.
>
> Should the CB_OFFLOAD request arrive before the client has
> fully processed the COPY response, the client will not yet
> recognize the operation. Client implementers might choose to
> implement a cache of CB_OFFLOAD requests in the expectation
> of a matching COPY response arriving imminently.
>
> However, this design can enable a broken or malicious server
> to overrun a client's cache by sending CB_OFFLOAD requests
> with junk copy state IDs. A client that implements such a
> cache needs to manage the content and size of that cache to
> avoid a denial of service.
>
> Alternately, the client implementation could simply return
> NFS4ERR_DELAY when it does not recognize a copy state ID
> presented in a CB_OFFLOAD operation. This introduces an extra
> delay and round trip, but completely avoids an overrun of
> cached copy completions.

# IANA Considerations

This document requests no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

Special thanks to Olga Kornievskaia, Rick Macklem, and Dai Ngo for
their insights and work on implementations of NFSv4.2 COPY.

The author is grateful to Bill Baker, Greg Marsden, and Martin Thomson
for their input and support.

Special thanks to Transport Area Directors Martin Duke and
Zaheduzzaman Sarker, NFSV4 Working Group Chairs Chris Inacio and
Brian Pawlowski, and NFSV4 Working Group Secretary Thomas Haynes for
their guidance and oversight.

