---
docname: draft-westerlund-masque-connect-udp-ecn-latest
title: ECN support for Connect-UDP
v: 3
submissiontype: IETF
category: std
area: WIT
wg: MASQUE
number:
date:
consensus: true
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

HTTP's Extended Connect-UDP protocol enables a client to proxy a UDP
flow from the HTTP server towards a specified target IP address and
UDP port. Both QUIC and RTP are transport protocols that uses UDP and
have support for using ECN and providing the necessary feedback. Thus,
there exist a benefit to enable an end-to-end QUIC or RTP transport
protocol connection/session to be able to use ECN. This document
specifies how ECN can be supported through an extension to the
Connect-UDP protocol for HTTP.


--- middle

# Introduction

Connect-UDP as currently defined limits the ECN {{RFC3168}} exchange between the
HTTP server and the target. There is no support for carrying the ECN
bits between the HTTP Connect-UDP client and the HTTP server proxying
the UDP flow. Thus, it is not possible to establish the end-to-end ECN
flow of information necessary to make either ECN classic {{RFC3168}}
or L4S {{RFC9484}} function.

To enable ECN usage end-to-end for transport protocols using UDP and
being proxied using the Connect-UDP extended connect over HTTP
additional extensions are needed. This document specifies the
necessary extension, defining two methods, one possible having no
additional overhead by encoding in the context ID, the other carrying
both DSCP and ECN. For these negotiation is specified and a new context that
carries the DSCP and ECN bits and the UDP payload instead of the context
defined in {{RFC9298}}.

An alternative to using this extension would be to use Connect-IP
{{RFC9484}}, however that results in that one carries a full IP header
between the HTTP client and HTTP server. That results in significantly
more overhead compared to this extension that adds zero or one byte
for containing both the DSCP and ECN bits.

To define a solution that can be combined with other contexts and
avoid having to be redefined for each combination with other
extensions this defines context IDs that indicates that it carries
the ECN and DSCP and then is followed by another Context ID. The
ECN encoding in the context ID defines three additional Context ID
that can be bound to following Context ID to indicate ECT(0), ECT(1),
or CE.

An endpoint should not enable both of the alternatives in this
document as that would lead to confusion on when to use the DSCP
carrying extension.

# Conventions

{::boilerplate bcp14}

# ECN encoded in Context ID {#sec-CID-ECN}

For a likely zero overhead encoding of the ECN bits, the ECN bits
can be indicated by using different context ID values followed by
the context that would otherwise have been used, often the basic
UDP payload context (CID=0) as defined by {{RFC9298}}.

The core of this solution is to define for a Connect-UDP stream define
additioanl Context IDs (A, B, and C) that are used to indicate the
ECN values that are not Not-ECT, i.e. ECT(1), ECT(0), CE {{RFC3168}}. This idea is shown in {{ECN-Encoding-Table}}.

| CID Value | ECN bit | ECN Value  | Original CID Value
| 0 | 0b00 | Not-ECT | 0
| A | 0b01 | ECT(1) | 0
| B | 0b10 | ECT(0) | 0
| C | 0b11 | CE | 0
{: #ECN-Encoding-Table title="ECN Encoding Table" cols="r l l l"}

No new CID value is defined to represent Not-ECT, as using a Context
value without this extension by default would be not-ECT. Then additionally
CIDs are defined to represent the combination of an ECN value other than
Not-ECT and the original CID. If the application used more CID values than
just zero, then additional CID values will have to be defined. This extension
is likely to result in three times additioanl CIDs will be needed in the
Connect-UDP stream. We expect that this will be acceptable in most cases
as a total of 64 CIDs can be encoded as a single byte, thus resulting
in no packet expansion. However for applications that has more than
16 original CIDs (including zero) it is possible to consider using the
ECN and DSCP context {{sec-DSCP-ECN}} which only double the number of
CIDs.

A endpoint enabling this extension MUST define all three of the
Not-ECT ECN values. This despite that an ECN application working as
intened would expect to see only the one the one ECT values the
endpoint uses and the CE code. However, if there are transmission
errors or remarking errors in the network also the other ECT codepoint
as well as Not-ECT may be needed to transmitt.

Negotiation of the CID values to use are defined using both
HTTP headers and Capsulses in {{sec-DSCP-ECN}}.


# ECN and DSCP enabled UDP Proxying Payload {#sec-DSCP-ECN}



The HTTP Datagram Payload format is defined in {{RFC9484}} as depicted below.

~~~ ascii-art
UDP Proxying HTTP Datagram Payload {
  Context ID (i),
  UDP Proxying Payload (..)
}
~~~
{: #dgram-format title="ECN enabled UDP Proxying HTTP Datagram Format"}



The ECN carrying UDP Proxying Payload is defined in the following way:

~~~ ascii-art
ECN carrying UDP Proxying Payload {
  DSCP(6),
  ECN(2),
  UDP Proxying Payload (..) # Payload per another Context ID definition
}
~~~
{: #UDP-Proxying-Payload title="ECN carrying UDP Proxying Payload"}

DSCP: six bits currently reserved, SHALL be set to zero (0) on
    transmission, and ignored on reception.

ECN: A two bit field carrying the IP ECN bits as defined by Section 5
    of {{RFC3168}} to be set or as received on the IP/UDP packet
    carrying the UDP Datagram payload included.

UDP Proxying Payload: Another UDP Proxying Payload following the ECN
    carrying byte. This uses another Context ID as negotiated,
    e.g. Context ID 0.  Thus enabling this byte to be combined with
    any other payload.

This format will use a negotiated context ID that MUST be non-zero. It also
MUST negotiate what the context ID of the UDP Proxying payload included.


# Negotiating ECN Usage {#sec-neg-ECN}


ECN-Context-ID is an Structured Header Field {{RFC9651}}. Its value
is a List consisting of zero or more Inner Lists, where the Inner List
contains two Integer Items. The Integer Items MUST be non-negative as
they are context Identifiers as defined by {{RFC9298}}. The first
Integer Item is the Context ID being defined, the second Context ID
is the Context ID for the payload following the ECN byte.

Note that Context Identifiers are defined as QUIC varints, see section
16 of {{RFC9000}} and do support value up to
4,611,686,018,427,387,903, which is larger than what an Structure
Header Integer support, which is limited to 999,999,999,999,999. We
forsee no issues with this limitations as context identifiers should
primiarly be using the single byte representation for efficiency
reason, i.e. Context Identifers should rarely be above the value 63.

When the header is included in a Extended Connect Request it indicate
first of all support for this ECN extension. Secondly is defines one
or more context IDs that is defined for requestor to responder direction.

The responder MAY if the request include the ECN-Context-ID header include
the header in a response. If included in a response it defines the Context ID
used in the responder to requestor direction.



~~~ ascii-art
ECN-Context-ID: (1,4), (2,5), (3,0)
~~~
{: #ECN-Context-ID-example title="Example of ECN-Context-ID header"}

The above example indicate supoprt of the ECN byte context and defines
three Context IDs, Context ID=1 is combined with 4, ID=2 with Context ID 5
and ID=3 with the default ID=0 as defined in {{RFC9298}}, i.e. a plain
UDP Payload.


# Tunnels and DSCP and ECN marking interactions

## Tunnel Endpoint Marking

The Tunnel Endpoint when receiving an IP/UDP packet that is belonging
to an Connect-UDP request where the ECN+DSCP extension is enabled
copies the the six DSCP bits and the two ECN field bits from the IP
header to the DSCP and ECN fields respectively in the HTTP datagram
payload using the relevant Context ID. If using the ECN only
extension the two ECN bits in the incoming IP/UDP packet are used to select
which CID value should be used.

For the DSCP+ECN extension a Tunnel endpoint on egress copies the DSCP
and ECN extension field value into the IP/UDP packet it creates for
this UDP Proxying payload. For the ECN extension the CID value is
used to determine which value to write in the outgoing IP/UDP packets
ECN field.

An Tunnel endpoint which is unable to read or set the ECN Field SHALL NOT
enable the ECN extension.

## DSCP Remarking Considerations

The tunnel may interconnect two different adminstrative domains
that uses the DSCP values differently. Thus, the endpoints likely
need to perform remarking of the DSCP field values the same as a
inter domain router would. Depending on use cases and deployment
the HTTP client can be in different actual network domains with
different DSCP usages. An HTTP server that based on user identification
connects the HTTP client to different network domains behind it may
also need to support multiple external domains.

The above complications in handling DSCP makes it impossible
to provide a standarized remarking instruction. Instead the
deployment will have to define if remarking is handled by the HTTP
server, the HTTP client, or both considering the tunnel a specific
network domain in itself.


## Tunnel Transport Connection ECN Interactions

The primary goal of the ECN extension is to enable ECN usage between
the proxy and the target and have the end-to-end transport react to
that ECN. However, there exist different potential models for how to
provide ECN interactions for the tunnel, i.e. between HTTP client and
sever. Which to do depends on how the tunnel is configured and what
other support one have implemented for the Connect-UDP protocol.

For HTTP tunnels not using HTTP/3 {{RFC9114}}, HTTP/3 using data
streams, or HTTP/3 with datagrams but not disabling congestion
control, the tunnel will consist of one or possibly several chained
congestion controlled transport connections. In this case ECN may be
enabled for each underlying transport connection
independently. However, it in this case it is not possible to have a
one-to-one marking between the lower layer ECN marking and the
tunneled HTTP datagrams and avoid reacting both on the tunnel
transport and in the end-to-end. Instead the only real choice to have
each HTTP layer hop run an ECN marking AQM goverened by what it can
transmit over the transport connection that marks the ECN field in the
HTTP datagram or drops the HTTP datagram when a queue builds. In other
words each HTTP transport connection is treated as one IP link on the
end-to-end chain.

For tunnels using HTTP/3 and Datagram and where the QUIC connection is
disabling congestion control on the packets containing HTTP datagrams,
as discsussed in Section 6 of {{RFC9298}}, then the ECN marking on the
tunneled packets can be propogated between the IP packet of the
transport connection and the end-to-end packet. This reprensts a
specific implementation of IP in IP Tunnels with Tightly coupled Shim
Headers as discussed in {{RFC9601}}. This is implemented as
Feed-Forward-and-Up as discussed in {{RFC9599}}, and MUST use the the
normal mode on tunnel ingress, and follow the specified default
behavior on egress as defined by {{RFC6040}}.

## Tunnel Transport Connection DSCP Interactions

For HTTP tunnels not using HTTP/3 {{RFC9114}}, HTTP/3 using data
streams, or HTTP/3 with datagrams but not disabling congestion
control, the tunnel will consist of one or possibly several chained
congestion controlled transport connections. These transport
connections will only be able to use a single DSCP code point to avoid
inconsistent network treatment that might confuse the congestion
controller and retransmission mechanism. Thus, even if the tunneled
packets are using different DSCP, the transport connection will have
to settle for using a single DSCP of suitable value. This unless the
Multipath extension for QUIC {{I-D.ietf-quic-multipath}} is used,
where each path can have a different DSCP value. In this later case
packets with different DSCP values could be mapped to different paths
with the approparite network treatment as indicated by the DSCP value.

For tunnels using HTTP/3 and Datagram and where the QUIC connection is
disabling congestion control on the packets containing HTTP datagrams,
as discsussed in Section 6 of {{RFC9298}}, the QUIC packets could be
marked using the most suitable DSCP value based on the encapsualted
packet.  In cases where the tunnel connection is sent in a different
network domain than the one the tunneled packet was received on, a
suitable remapping needs to occur to the domain the tunnel packet will
be sent to. The HTTP tunnel MUST not coalesce different tunneled
payloads which are not mapped to the same DSCP in the same QUIC packet.



## QUIC Aware Forwarding

A HTTP endpoint that supports this extension and QUIC Aware Forwarding
{{I-D.ietf-masque-quic-proxy}} MUST preserve ECN markings on forwarded
packets in both directions, to ensure ECN functions end-to-end. Using
this extension in combination with the QUIC aware forwarding instead
of only QUIC aware forwarding also ensure that one don't get ECN black
holes on some packets, like long header packets or for packets sent at
any point when the QUIC Aware forwarding isn't yet established for
short header packets. Thus supporting both ensures a consistent ECN
experience.



# Open Issues



# IANA Considerations

## HTTP Field Name

IANA is request to register one new permanent Field name in the
Hypertext Transfer Protocol (HTTP) Field Name Registry (At time of
writing residing at:
https://www.iana.org/assignments/http-fields/http-fields.xhtml).

   Field Name: ECN-Context-ID

   Status: Permanent

   Structured Type: List

   Reference: [RFC-TO-BE]

## HTTP Capsule Type




# Acknowledgments

This draft takes insperation from David Schinazi's An ECN Extension to
Connect-UDP {{I-D.schinazi-masque-connect-udp-ecn}}.

--- back
