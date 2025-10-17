---
docname: draft-westerlund-masque-connect-udp-ecn-dscp-latest
title: ECN and DSCP support for HTTPS's Connect-UDP
v: 3
submissiontype: IETF
category: std
area: WIT
wg: MASQUE
number:
date:
consensus: false
venue:
  group: "MASQUE"
  type: "Working Group"
  mail: "masque@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/masque/"
  github: "gloinul/masque-ecn"
  latest: "https://gloinul.github.io/masque-ecn/#go.draft-westerlund-masque-connect-udp-ecn.html"
keyword:
  - quic
  - http
  - datagram
  - udp
  - proxy
  - tunnels
  - quic in quic
  - turtles all the way down
  - masque
  - http-ng
author:
-
   ins:  M. Westerlund
   name: Magnus Westerlund
   org: Ericsson
   email: magnus.westerlund@ericsson.com

informative:
   RFC8311:
   RFC9484:
   I-D.schinazi-masque-connect-udp-ecn:
   I-D.ietf-quic-multipath:

normative:
   I-D.ietf-masque-quic-proxy:
   RFC2474:
   RFC2475:
   RFC3168:
   RFC6040:
   RFC9000:
   RFC9114:
   RFC9298:
   RFC9297:
   RFC9330:
   RFC9599:
   RFC9601:
   RFC9651:


--- abstract

HTTP's Extended Connect's Connect-UDP protocol enables a client to
proxy a UDP flow from the HTTP server towards a specified target IP
address and UDP port. QUIC and Real-time transport protocol (RTP) are examples
of transport protocols that use UDP and support Explicit Congestion
Notification (ECN) and provide the necessary feedback. This document specifies
how ECN and DSCP can be supported through an extension to the Connect-UDP
protocol for HTTP.


--- middle

# Introduction

Connect-UDP, as currently defined, limits the Explicit Congestion Notification
(ECN) {{RFC3168}} exchange between the HTTP server and the target. There is no
support for carrying the ECN bits between the HTTP Connect-UDP client and the
HTTP server proxying the UDP flow. Thus, it is not possible to establish the
end-to-end ECN information flow necessary to support either classic ECN
{{RFC3168}} or L4S {{RFC9484}}.

Diffserv {{RFC2475}} enables differential network treatment of packets.
Connect-UDP, as currently defined, lacks support for carrying the DSCP
field {{RFC2474}} through the tunnel.

This document specifies two Connect-UDP extensions that enable end-to-end ECN
and DSCP for proxied connections: an ECN-zero-bytes extension, which adds no
overhead by encoding the ECN value directly into the Context ID; and an
ECN/DSCP extension, which carries both DSCP and ECN in the HTTP Datagram
payload. For these two extensions, this document specifies negotiation and
defines a new Datagram context that carries the DSCP and ECN bits and the UDP
payload, replacing the context defined in {{RFC9298}}.

An alternative to this extension is Connect-IP {{RFC9484}}; however, it carries
a full IP header between the HTTP client and server, resulting in significantly
more overhead than this extension, which requires zero or one byte to carry
both the DSCP and ECN bits.

To define a solution that can be combined with other extensions, and thus other
contexts, without redefining each combination, the extensions defined in this
document indicate not only the ECN and DSCP values but also the next Context ID.
The ECN-zero-bytes extension defines three additional Context ID values that are
bound to an HTTP Datagram payload identifying the Context ID and indicate whether
the packet was marked with ECT(0), ECT(1), or CE, respectively.

The extensions are defined such that they allow clients to optimistically start
sending UDP packets in HTTP Datagrams, i.e. before receiving the response to its
UDP proxying request, as described in {{Section 5 of RFC9298}}.

An endpoint should not enable both extensions defined in this document, as that
would lead to confusion if both extensions indicate different ECN values.

# Conventions

{::boilerplate bcp14}

# ECN-zero-byte Extension {#sec-CONTEXT-ECN}

For a zero-overhead encoding, the ECN bits can be indicated by using different
Context IDs. At the same time, the Context ID indicates which Context ID would
otherwise have been used to identify the structure of the rest of the HTTP
payload, which we call the payload-identifying context ID, or short payload
ID. Often this is the basic UDP-payload context (Context ID 0) as defined by
{{RFC9298}}.

The core of this solution is to define additional Context IDs (A, B, and C) for
a Connect-UDP stream that indicate ECN values other than Not-ECT, i.e., ECT(1),
ECT(0), or CE {{RFC3168}}. This idea is shown in {{ECN-Encoding-Table}}.

| Context ID Value | ECN bit | ECN Value  | Payload ID
| 0 | 0b00 | Not-ECT | 0
| A | 0b01 | ECT(1) | 0
| B | 0b10 | ECT(0) | 0
| C | 0b11 | CE | 0
{: #ECN-Encoding-Table title="ECN Encoding Table" cols="r l l l"}

No new Context ID value is defined to represent Not-ECT, since using a Context
ID without this extension would, by default, imply Not-ECT. Additionally,
Context IDs are defined to represent the combination of an ECN value other than
Not-ECT and the payload-identifying Context ID. If an application uses more
Context ID values than just zero, additional Context IDs must be defined.

This extension results in four times as many Context IDs within a single
Connect-UDP stream. We expect that this is acceptable in most cases, as a total
of 64 Context IDs can be encoded in a single byte, thus resulting in no packet
expansion. However, for applications that have more than 16 original Context IDs
(including zero), it is recommended to use the ECN/DSCP extension
{{sec-DSCP-ECN}}, which only doubles the number of Context IDs but requires an
additional byte in the payload.

An endpoint enabling this extension MUST define all three ECN values, even if
the ECN-enabled application expects that only one ECT value (and CE) is used.
This is because of transmission errors or erroneous remarking in the network,
where the other ECT codepoint, as well as Not-ECT, may be observed.

Negotiation of the conetxt ID values is defined using both
HTTP headers and capsulses in {{sec-neg-ECN-Zero}}.


# DSCP/ECN Extension {#sec-DSCP-ECN}

The HTTP Datagram Payload format is defined in {{RFC9484}} as depicted below.

~~~ ascii-art
UDP Proxying HTTP Datagram Payload {
  Context ID (i),
  UDP Proxying Payload (..)
}
~~~
{: #dgram-format title="ECN enabled UDP Proxying HTTP Datagram Format"}

For the ECN/DSCP extension, the ECN/DSCP UDP Proxying Payload is defined in the following way:

~~~ ascii-art
ECN/DSCP UDP Proxying Payload {
  DSCP(6),
  ECN(2),
  UDP Proxying Payload (..) # Payload per another Context ID definition
}
~~~
{: #UDP-Proxying-Payload title="ECN/DSCP UDP Proxying Payload"}

DSCP: A six bit field carrying the Differentiated Service Code Point
    as defined by {{RFC2474}} associated with this UDP Proxying Payload.

ECN: A two bit field carrying the IP ECN bits as defined by Section 5
    of {{RFC3168}} to be set or as received on the IP/UDP packet
    carrying the UDP Datagram payload included.

UDP Proxying Payload: Another UDP Proxying Payload following the ECN
    carrying byte. This uses another Context ID as negotiated,
    e.g. Context ID 0.  Thus enabling this byte to be combined with
    any other payload.

This format used a negotiated context ID that MUST be non-zero. It
MUST also negotiate the payload-identifying context ID.


# Negotiating Extensions Usage {#sec-neg-ECN}

This section defines capability negotation and Context ID
configuration for the two defined extensions.

Note that Context Identifiers are defined as QUIC varints (see Section 16 of
{{RFC9000}}) and support values up to 4,611,686,018,427,387,903, which is
larger than what a Structure Header Integer supports (limited to
999,999,999,999,999). We foresee no issues with this limitation, as Context
Identifiers should primarily use the single-byte representation for efficiency,
i.e., they should rarely exceed 63.


## ECN-zero-byte Extension {#sec-neg-ECN-Zero}

To use the ECN-zero-byte extension three Context IDs need to be configured
relative to the Context ID that identifies the actual HTTP Datagram payload.

A configuration of ECN signaling is represented by a four-tuple with the
following format:

~~~ ascii-art
ECN_CONTEXT_ASSIGNMENT {
  ECT_1_CONTEXT (i),
  ECT_0_CONTEXT (i),
  CE_CONTEXT (i),
  PAYLOAD_CONTEXT (i)
}
~~~
{: #ECN-CAP-CONTEXT-Format title="ECN CONTEXT ASSIGNMENT Format"}

ECT_1_CONTEXT:
: The Context ID used to indicate the payload was marked with ECN value ECT(1).

ECT_0_CONTEXT:
: The Context ID used to indicate the payload was marked with ECN value ECT(0).

CE_CONTEXT:
: The Context ID used to indicate the payload was marked with ECN value CE.

PAYLOAD_CONTEXT:
: The payload-identifiying context ID indicating the actual encoding of the
start of the HTTP Datagram payload.


### HTTP Structured field

ECN-Context-ID is a Structured Header Field {{RFC9651}}. Its value is a List
consisting of zero or more Inner Lists, where the Inner List contains four
Integer Items. The Integer Items MUST be non-negative, as they are context
identifiers defined in {{RFC9298}}. The four Context IDs are those defined in
{{ECN-CAP-CONTEXT-Format}}, in that order.

When the header is included in an Extended Connect request, it indicates, first
of all, support for this ECN extension. Secondly, it may define one or more
4-item inner lists of Context IDs for the requestor-to-responder direction.
If no 4-item inner lists of Context IDs are included, then this header only
indicates support for the extension, and the Context IDs MAY be signaled using
capsules.

When the request includes the ECN-Context-ID header, the responder MAY include
this header in the response. If included, it defines the Context ID used in the
responder-to-requestor direction.

The following example indicates support for ECN-zero-byte-extension and
defines two sets of Context IDs: ID=5, 6, 7 (ECT(1), ECT(0), CE) combined with
Context ID 4; and ID=1, 2, 3 combined with the default ID=0 as defined in
{{RFC9298}}, i.e., a plain UDP payload.

~~~ ascii-art
ECN-Context-ID: (5,6,7,4), (1,2,3,0)
~~~
{: #ECN-Context-ID-example title="Example of ECN-Context-ID header"}


### ECN Context ID Assignment Capsule

The ECN_CONTEXT_ASSIGN capsule is used to assign Context ID values to the
ECN-zero-byte extension.

~~~ ascii-art
ECN_CONTEXT_ASSIGN Capsule {
  Type (i) = TBA_2
  Length (i),
  ECN_CONTEXT_ASSIGNMENT (..) ...,
}
~~~
{: #ECN-CAP-Format title="ECN_CONTEXT_ASSIGN Capsule Format"}

Type and Length as defined by Section 3.2 of the HTTP Capsule specification
{{RFC9297}}. The capsule value is the ECN_CONTEXT_ASSIGNMENT defined above in
{{ECN-CAP-CONTEXT-Format}}. Thus, the capsule value consists of zero or more
ECN_CONTEXT_ASSIGNMENT four-tuples.


## ECN/DSCP extension

This section defines the negotiation of the ECN/DSCP extension defined in
{{sec-DSCP-ECN}}. It defines both an HTTP header field (DSCP-ECN-Context-ID)
that can be included in the Extended Connect request, and a capsule.


### HTTP Structured Header

DSCP-ECN-Context-ID is a Structured Header Field {{RFC9651}}. Its
value is a List of zero or more Inner Lists, where each Inner List
contains two Integer Items. The Integer Items MUST be non-negative, as
they are context identifiers defined by {{RFC9298}}. The first Integer
Item is the Context ID being defined, and the second Integer Item is
the Context ID for the payload following the ECN/DSCP byte.

When the header is included in an Extended Connect request, it indicates, first
of all, support for the ECN/DSCP extension. Secondly, it may define one or more
context ID pairs for the requestor-to-responder direction. If no context ID
pairs are included, then this header only indicates support for the extension,
and it may be configured using capsules.

When the request includes the DSCP-ECN-Context-ID header, the responder MAY
include this header in the response. If included, it defines the Context ID
used in the responder-to-requestor direction

The following example indicates supoprt of the ECN/DSCP extension and defines
three Context IDs: Context ID 1 combined with Context ID 4, Context ID 2
combined with Context ID 5, and Context ID 3 combined with the default Context
ID 0 as defined in {{RFC9298}}, i.e., a plain UDP payload.

~~~ ascii-art
DSCP-ECN-Conext-ID: (4,1), (5,2), (3,0)
~~~
{: #DSCP-ECN-Context-ID-example title="Example of ECN-Context-ID header"}


### DSCP/ECN Context ID Assignment Capsule

The DSCP_ECN_CONTEXT_ASSIGN capsule is used to assign context ID values for the
DSCP/ECN extension.

~~~ ascii-art
DSCP_ECN_CONTEXT_ASSIGN Capsule {
  Type (i) = TBA_2
  Length (i),
  CONTEXT ASSIGNMENT (..) ...,
}
~~~
{: #DSCP-ECN-CAP-Format title="DSCP_ECN_CONTEXT_ASSIGN Capsule Format"}

Type and Length as defined by the HTTP Capsule specification in {{Section
3.2 of RFC9297}}. The capsule value is defined below.

~~~ ascii-art
CONTEXT ASSIGNMENT {
  ASSIGNED_CONTEXT (i),
  NEXT_PAYLOAD_CONTEXT (i)
}
~~~
{: #DSCP-ECN-CAP-CONTEXT-Format title="CONTEXT ASSIGNMENT Format"}

The capsule value consists of zero or more CCONTEXT ASSIGNMENT pair values.
Each pair consists of these two fields:

ASSIGNED_CONTEXT: : The context ID identifying that the indicated HTTP datagram
payload starts with the ECN/DSCP UDP Proxying Payload.

NEXT_PAYLOAD_CONTEXT:
: The context ID identifying the following payload in each
HTTP datagram after the ECN/DSCP UDP Proxying Payload.

This capsule is sent by either endpoints to configure or extend the
configuration of context IDs. The receiving HTTP
endpoint MUST send back its corresponding DSCP_ECN_CONTEXT_ASSIGN capsule,
which may be empty if the peer does not intend to provide any context IDs.


# Tunnels and DSCP and ECN marking interactions

## Tunnel Endpoint Marking

The Tunnel Endpoint, when receiving an IP/UDP packet belonging to a Connect-UDP
request with the ECN/DSCP extension enabled, copies the six DSCP bits and the
two ECN bits from the IP header into the DSCP and ECN fields, respectively, in
the HTTP Datagram payload using the relevant Context ID. When using the ECN-only
extension, the two ECN bits in the incoming IP/UDP packet are used to select the
appropriate Context ID.

For the ECN/DSCP extension, the Tunnel Endpoint on egress copies the DSCP and
ECN extension field values into the IP/UDP packet it creates for this UDP
Proxying payload. For the ECN extension, the Context ID value is used to
determine which value to write in the outgoing IP/UDP packet’s ECN field.

A Tunnel endpoint which is unable to read or set the ECN Field SHALL NOT
enable the ECN extension.


## DSCP Remarking Considerations

The tunnel may interconnect two different administrative domains that use
DSCP values differently. Thus, the endpoints likely need to perform remarking
of DSCP field values, similar to what an inter-domain router would. Depending on
use cases and deployment, the HTTP client can be in different network domains
with different DSCP usages. An HTTP server that, based on user identification,
connects the HTTP client to different network domains behind it may also need
to support multiple external domains.

The above complications in handling DSCP make it impossible to provide a
standardized remarking instruction. Instead, the deployment will have to define
whether remarking is handled by the HTTP server, the HTTP client, or both,
considering the tunnel a specific network domain in itself.


## Tunnel Transport Connection ECN Interactions and Congestion Control

The primary goal of the ECN extension is to enable ECN usage between the proxy
and the target and to have the end-to-end transport react to that ECN. However,
different potential models exist for providing ECN interactions for the tunnel,
i.e., between the HTTP client and server. The choice depends on how the tunnel
is configured and what additional support has been implemented for the
Connect-UDP protocol.

For HTTP tunnels (not using HTTP/3 {{RFC9114}}, HTTP/3 using data streams, or
HTTP/3 with datagrams but with congestion control enabled) the tunnel will
consist of one or possibly several chained congestion-controlled transport
connections. In this case, ECN may be enabled independently on each underlying
transport connection. However, in this scenario, it is not possible to have a
one-to-one correspondence between lower-layer ECN markings and tunneled HTTP
datagrams, nor to avoid reacting at both the tunnel-transport layer and the
end-to-end layer. Instead, the only practical choice is to have each HTTP-layer
hop run an ECN-marking AQM governed by its transport connection, which marks
the ECN field in the HTTP datagram or drops the datagram when a queue builds.
In other words, each HTTP transport connection is treated as a single IP link
in the end-to-end chain.

For tunnels using HTTP/3 with datagrams, where the QUIC connection disables
congestion control on packets containing HTTP datagrams as discussed in Section
6 of {{RFC9298}}, the ECN marking on tunneled packets can be propagated between
the IP packet of the transport connection and the end-to-end packet. This
represents a specific implementation of IP-in-IP tunnels with tightly coupled
shim headers as discussed in {{RFC9601}}. It is implemented as
Feed-Forward-and-Up as discussed in {{RFC9599}}, and MUST use the normal mode on
tunnel ingress and follow the specified default behavior on egress as defined in
{{RFC6040}}.


## Tunnel Transport Connection DSCP Interactions

For HTTP tunnels not using HTTP/3 {{RFC9114}}, HTTP/3 using data streams, or
HTTP/3 with datagrams but without disabling congestion control, the tunnel will
consist of one or possibly several chained congestion-controlled transport
connections. These transport connections can use only a single DSCP code point
to avoid inconsistent network treatment that might confuse the congestion
controller and retransmission mechanism. Thus, even if the tunneled packets use
different DSCP values, the transport connection must settle on a single,
suitable DSCP value. However, if the QUIC multipath extension
{{I-D.ietf-quic-multipath}} is used, each path can have a different DSCP value.
In this latter case, packets with different DSCP values can be mapped to
different paths with the appropriate network treatment as indicated by their
DSCP values.

For tunnels using HTTP/3 with datagrams and where the QUIC connection disables
congestion control on packets containing HTTP datagrams, as discussed in
Section 6 of {{RFC9298}}, the QUIC packets can be marked using the most
suitable DSCP value based on the encapsulated packet. In cases where the tunnel
connection is sent into a different network domain than the one on which the
tunneled packet was received, a suitable remapping must occur for the domain to
which the tunnel packet will be sent. The HTTP tunnel MUST NOT coalesce
different tunneled payloads that are not mapped to the same DSCP in a single
QUIC packet.

## QUIC Aware Forwarding

An HTTP endpoint that supports this extension and QUIC Aware Forwarding
{{I-D.ietf-masque-quic-proxy}} MUST preserve ECN markings on forwarded packets
in both directions to ensure end-to-end ECN functionality. Using this extension
in combination with QUIC Aware Forwarding, rather than relying solely on the
latter, also ensures that ECN black holes do not occur, for example, on
long-header packets or packets sent before the QUIC Aware Forwarding path is
established for short-header packets. Thus, supporting both provides a
consistent ECN experience.


# IANA Considerations

## HTTP Field Names

IANA is request to register two new permanent Field name in the
Hypertext Transfer Protocol (HTTP) Field Name Registry (At time of
writing residing at:
https://www.iana.org/assignments/http-fields/http-fields.xhtml).

### ECN-Context-ID

   Field Name: ECN-Context-ID

   Status: Permanent

   Structured Type: List

   Reference: [RFC-TO-BE]

### DSCP-ECN-Context-ID

   Field Name: DSCP-ECN-Context-ID

   Status: Permanent

   Structured Type: List

   Reference: [RFC-TO-BE]



## HTTP Capsule Type

IANA is reqeusted ot register two new HTTP Capsule Types in the
permanent range (0x00-0x3f).

### ECN_CONTEXT_ASSIGN

Value:
: TBA_1

Capsule Type:
: ECN_CONTEXT_ASSIGN

Status:
: permanent

Reference:
: RFC-TO-BE

Change Controller:
: IETF

Contact:
: MASQUE Working Group masque@ietf.org

Notes:
: None


### DSCP_ECN_CONTEXT_ASSIGN

Value:
: TBA_2

Capsule Type:
: DSCP_ECN_CONTEXT_ASSIGN

Status:
: permanent

Reference:
: RFC-TO-BE

Change Controller:
: IETF

Contact:
: MASQUE Working Group masque@ietf.org

Notes:
: None


# Acknowledgments

This draft takes insperation from David Schinazi's An ECN Extension to
Connect-UDP {{I-D.schinazi-masque-connect-udp-ecn}}.

--- back
