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
   RFC2474:
   RFC8311:
   RFC9484:
   I-D.schinazi-masque-connect-udp-ecn:
   I-D.ietf-quic-multipath:

normative:
   I-D.ietf-masque-quic-proxy:
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
address and UDP port. Both QUIC and RTP are transport protocols that
uses UDP and have support Explicit Congestion Notification (ECN) by providing the necessary
feedback. Thus, it is benefitial to enable an end-to-end QUIC connections or
Real-time transport protocol (RTP) sessions to use ECN. This
document specifies how ECN and DSCP can be supported through an
extension to the Connect-UDP protocol for HTTP.


--- middle

# Introduction

Connect-UDP as currently defined limits the Explicit Congestion Notification (ECN) {{RFC3168}} exchange between the
HTTP server and the target. There is no support for carrying the ECN
bits between the HTTP Connect-UDP client and the HTTP server proxying
the UDP flow. Thus, it is not possible to establish the end-to-end ECN
flow of information necessary to support either classic ECN {{RFC3168}}
or L4S {{RFC9484}}.

Diffserv {{RFC2475}} enables differential network treatment of
packets. Connect-UDP as currently defined lacks support for carrying
the value of the DSCP field {{RFC2474}} through the tunnel.

As such additional extensions are needed to enable DSCP and ECN usage end-to-end for UDP-based transport protocols
that are proxied by Connect-UDP extended connect over HTTP. This document specifies two such
extensions: The ECN-zero-bytes extension without additional overhead by encoding the ECN value directly into the context ID;
and the ECN/DSCP extension that
carries both DSCP and ECN in the HTTP Datagram payload. For these two
extensions negotiation is specified and a new Datagram context that
carries the DSCP and ECN bits and the UDP payload instead of the
context defined in {{RFC9298}}.

An alternative to using this extension would be to use Connect-IP
{{RFC9484}}, however, that that approach carries a full IP header
between the HTTP client and HTTP server which results in significantly
more overhead compared to these extensions that add zero or one byte
for ECN bits and DSCP values.

To define a solution that can be combined with other extension and as
such other contexts instead of redefining each combination with other
extensions, the extensions defined in this document indicates not only the
ECN and DSCP values but also the next following Context ID. The ECN-zero-bytes
extension defines three additional Context ID
values that are bound to a HTTP Datagram payload identifying Context
ID and indicates if the packet was marked with ECT(0), ECT(1), or CE
respectively.

An endpoint should not enable both of the extension defined in this
document as that would lead to confusion if both extension would indicate
a different ECN value.

# Conventions

{::boilerplate bcp14}

# ECN-zero-byte Extension {#sec-CID-ECN}

For a zero overhead encoding, the ECN bits
can be indicated by using different context ID.
At the same time the context ID indicates what context ID
would otherwise have been used to identify the structure of the rest of the HTTP payload,
which we call the payload-identifying conext ID or short payload ID.
Often this isthe basic
UDP payload context (context ID=0) as defined by {{RFC9298}}.

The core of this solution is to define for a Connect-UDP stream
additioanl Context IDs (A, B, and C) that are used to indicate the ECN
values that are not Not-ECT, i.e. ECT(1), ECT(0), CE {{RFC3168}}. This
idea is shown in {{ECN-Encoding-Table}}.

| Context ID Value | ECN bit | ECN Value  | Payload ID
| 0 | 0b00 | Not-ECT | 0
| A | 0b01 | ECT(1) | 0
| B | 0b10 | ECT(0) | 0
| C | 0b11 | CE | 0
{: #ECN-Encoding-Table title="ECN Encoding Table" cols="r l l l"}

No new context ID value is defined to represent Not-ECT, as using a Context
value without this extension by default would be not-ECT.
Additionally context IDs are defined to represent the combination of an ECN
value other than Not-ECT and the actual payload-identifying context
ID. If the application uses more context ID values than just zero,
additional context ID values need to be defined.

This extension results in four times the number of conext IDs within one
Connect-UDP stream. We expect that is acceptable in most
cases as a total of 64 context IDs can be encoded as a single byte, thus
resulting in no packet expansion. However for applications that have
more than 16 original context IDs (including zero) it is recommened to
use the ECN/DSCP extension {{sec-DSCP-ECN}} which only doubles the
number of context IDs but requires an additional byte in the payload.

A endpoint enabling this extension MUST define all three of the
Not-ECT ECN values, even if the ECN-enables application
expects that only one ECT value (and CE) is used.
This is because of transmission errors or erronous
remarking in the network where also the other ECT codepoint as well as
Not-ECT may observed.

Negotiation of the conetxt ID values is defined using both
HTTP headers and capsulses in {{sec-DSCP-ECN}}.


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

DSCP: six bits currently reserved, SHALL be set to zero (0) on
    transmission, and ignored on reception.

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

Note that context IDs are defined as QUIC varints, see section
16 of {{RFC9000}} and do support value up to
4,611,686,018,427,387,903, which is larger than what an Structure
Header Integer support, which is limited to 999,999,999,999,999. We
forsee no issues with this limitations as context identifiers should
primiarly use the single byte representation for efficiency
reason, i.e. context IDs should rarely be above the value 63.


## ECN-zero-byte Extension

To use the ECN-zero-byte extension three context IDs need
to be signaled in relation to a payload-identifying context ID.

Each Actual configuration of an ECN signalled using a four tuple with the following format:

~~~ ascii-art
ECN_CONTEXT_ASSIGNMENT {
  ECT_1_CONTEXT (i),
  ECT_0_CONTEXT (i),
  CE_CONTEXT (i),
  PAYLOAD_CONTEXT (i)
}
~~~
{: #ECN-CAP-CID-Format title="ECN CONTEXT ASSIGNMENT Format"}

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

ECN-Context-ID is an Structured Header Field {{RFC9651}}. Its value
is a list consisting of zero or more inner lists, where an inner list
contains four integer items. The integer items MUST be non-negative as
they are context IDs as defined in {{RFC9298}}. The four context
IDs are the four defined in {{ECN-CAP-CID-Format}} in that order.

When the header is included in a Extended Connect Request, it indicates,
first of all, support for this ECN extension. Secondly, is may define
one or more 4-item inner lists of context IDs for the requestor-to-responder direction.
If no context ID 4-item inner lists are included then
this header only indicate support for the extension and the context IDs MAY be
signaled using capsules.

The responder MAY include the header in a response if the received request
included the ECN-Context-ID header. If included in a response it defines
the context IDs used in the responder-to-requestor direction.

The following example indicates support of the ECN-zero-byte extension
and defines two sets of context IDs, ID=5,6,7 (ECT(1), ECT(0),CE)
combined with context ID 4, and context ID=1,2,3 combined with
the default context ID 0 as defined in {{RFC9298}}, i.e. a plain UDP Payload.

~~~ ascii-art
ECN-Context-ID: (5,6,7,4), (1,2,3,0)
~~~
{: #ECN-Context-ID-example title="Example of ECN-Context-ID header"}


### ECN Context ID Assignment Capsule

The DSCP_ECN_CONTEXT_ASSIGN capsule is used to assign conext ID values for the
ECN-zero-byte extension.

~~~ ascii-art
ECN_CONTEXT_ASSIGN Capsule {
  Type (i) = TBA_2
  Length (i),
  ECN_CONTEXT ASSIGNMENT (..) ...,
}
~~~
{: #ECN-CAP-Format title="ECN_CONTEXT_ASSIGN Capsule Format"}

Type and Length are as defined by the HTTP Capsule specification
{{Section 3.2 of RFC9297}}. The capsule value is the ECN_CONEXT_ASSIGNMENT
defined above (see {{ECN-CAP-CID-Format}}). Thus, the capsule value consists
of zero or more Context Assignment 4-item lists.


## ECN/DSCP extension

This defines the negotiation of the ECN/DSCP extension defined in {{sec-DSCP-ECN}}. It defined both an HTTP
header field (DSCP-ECN-CONTEXT) that can be included in the
Extended Connect request as well as an capsule.


### HTTP Structured Header

DSCP-ECN-Context-ID is an Structured Header Field {{RFC9651}}. Its value
is a list consisting of zero or more inner lists, where an inner list
contains two integer items. The integer items MUST be non-negative as
they are context IDs as defined by {{RFC9298}}. The first
integer item is the context ID being defined, the second context ID
is the payload-identifying context ID for the payload following the ECN byte.

When the header is included in a Extended Connect Request it indicate,
first of all, support for the ECN/DSCP extension. Secondly, is defines
one or more context IDs that is defined for requestor to responder
direction. If no context ID pairs are included then this header only
indicate support for the extension and it may be configured using
capsules.

The responder MAY include the header in a response if the request include
the DSCP-ECN-Context-ID header. If included in a response it
defines thecContext IDs used in the responder-to-requestor direction.

The following example indicates supoprt of the ECN/DSCP extension and defines
three context IDs, context ID=1 is combined with 4, ID=2 with context ID 5
and ID=3 with the default ID=0 as defined in {{RFC9298}}, i.e. a plain
UDP payload.

~~~ ascii-art
DSCP-ECN-Conext-ID: (1,4), (2,5), (3,0)
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
{: #DSCP-ECN-CAP-CID-Format title="CONTEXT ASSIGNMENT Format"}

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

When a tunnel endpoint of a Connect-UDP connection receives an IP/UDP packet
and the ECN/DSCP extension is enabled, it copies the the six DSCP bits and the two ECN field bits from the IP
header to the DSCP and ECN fields in the HTTP datagram
payload using the respective Context ID. If the ECN-zero-byte
extension is used, the two ECN bits in the incoming IP/UDP packet are used to select
which context ID value should be used.

For the ECN/DSCP extension a tunnel egress endpoint copies the DSCP
and ECN extension field value into the IP/UDP packet it creates for
this UDP Proxying payload. For the ECN-zero-byte extension, the conext ID value is
used to determine which value to write in the outgoing IP/UDP packets
ECN field.

An Tunnel endpoint which is unable to read or set the ECN Field SHALL NOT
enable the ECN extension.


## DSCP Remarking Considerations

The tunnel may interconnect two different adminstrative domains
that use DSCP values differently. Thus, the endpoints likely
need to perform remarking of the DSCP field values, similar as an
inter domain router. Depending on use cases and deployment,
an HTTP client can be in different network domains with
different DSCP usages. An HTTP server that
connects the HTTP client to different network domains may
also need to support multiple external domains.

These complications in handling DSCP makes it impossible
to provide standarized remarking instructions. Instead the
deployment will have to define if remarking is handled by the HTTP
server, the HTTP client, or both considering the tunnel a specific
network domain in itself.


## Tunnel Transport Connection ECN Interactions and Congestion Control

The primary goal of the ECN extension is to enable ECN usage between
the proxy and the target and have the end-to-end transport react to
that ECN. However, there are different potential models for how to
provide ECN interactions for a tunnel, i.e. between the HTTP client and
sever. Which one to use depends on how the tunnel is configured and what
other support is implemented for the Connect-UDP protocol.

For HTTP tunnels (that are not using HTTP/3 {{RFC9114}}, HTTP/3 using data
streams, or HTTP/3 with datagrams with congestion
control), the each tunnel is one
congestion-controlled transport connections. In this case, ECN needs to be
enabled for each underlying transport connection
independently. That means it is not possible to have a
one-to-one marking between the lower layer ECN marking and the
tunneled HTTP datagrams to avoid reacting twice, on the tunnel
transport and in the end-to-end connection. Instead
each HTTP hop needs to run an AQM to set ECN CE mark or drop the HTTP datagram.
In other words, each HTTP transport connection is treated as one IP link on the
end-to-end chain.

For tunnels using HTTP/3 datagrams and where the QUIC connection
disabled congestion control,
as discsussed in Section 6 of {{RFC9298}}, the ECN marking on the
tunneled packets can be propogated between the IP packet of the
transport connection and the end-to-end packet. This represents a
specific implementation of IP-in-IP tunnels with tightly coupled shim
Headers as discussed in {{RFC9601}}. This is implemented as
Feed-Forward-and-Up as discussed in {{RFC9599}}, and MUST use the
normal mode on tunnel ingress, and follow the specified default
behavior on egress as defined by {{RFC6040}}.


## Tunnel Transport Connection DSCP Interactions

For HTTP tunnels not using HTTP/3 {{RFC9114}}, HTTP/3 using data
streams, or HTTP/3 with datagrams but not disabling congestion
control, the tunnel will consist of one or possibly several chained
congestion controlled transport connections. These transport
connections will only be able to use a single DSCP to avoid
inconsistent network treatment that might confuse the congestion
controller and retransmission mechanism. Thus, even if the tunneled
packets are using different DSCP values, the transport connection will have
to use a single DSCP; unless the
Multipath extension for QUIC {{I-D.ietf-quic-multipath}} is used,
where each path can have a different DSCP values. In this later case,
packets with different DSCP values could be mapped to different paths
with the approparite network treatment as indicated by the DSCP value.

For tunnels using HTTP/3 datagram and where the QUIC connection diabled congestion control,
as discsussed in {{Section 6 of RFC9298}}, the QUIC packets could be
marked using the most suitable DSCP value based on the encapsualted
packet.  In cases where the tunnel connection sents into a different
network domains than the one the tunneled packet was received on, a
suitable remapping needs to occur to the domain the tunnel packet will
be sent to. The HTTP tunnel MUST not coalesce different tunneled
payloads which are not mapped to the same DSCP value in the same QUIC packet.


## Usage with the QUIC Aware Forwarding extension

A HTTP endpoint that supports this extension and QUIC Aware Forwarding
{{I-D.ietf-masque-quic-proxy}} MUST preserve ECN markings on forwarded
packets in both directions, to ensure ECN functions end-to-end. Using
this extension in combination with the QUIC Aware forwarding instead
of only QUIC Aware forwarding also ensures that one does not get ECN black
holes on some packets, like long header packets or for packets sent at
any point when the QUIC Aware forwarding is not yet established for
short header packets. Thus supporting both ensures a consistent ECN
experience.


# Open Issues

- Is it the right mechanism to use an HTTP header with no
  configurations to indicate the capability to support
  later configuration using capsules?

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
