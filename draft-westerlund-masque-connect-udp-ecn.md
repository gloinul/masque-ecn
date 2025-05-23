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
or L4S {{RFC9484}}.

To enable ECN usage end-to-end for transport protocols using UDP and
being proxied using the Connect-UDP extended connect over HTTP
additional extensions are needed. This document specified the necessary
extension, defining a negotiation method and a new context that carries
the ECN bits and the UDP payload instead of the context defined in {{RFC9298}}.

An alternative to using this extension would be to use Connect-IP
{{RFC9484}}, however that results in that one carries a full IP header
between the HTTP client and HTTP server. That results in significantly
more overhead compared to this extension that adds one byte for containing
the ECN bits.

To define a solution that can be combined with other contexts and
avoid having to be redefined for each combination with other
extensions this defines a context ID that indicates that it carries
the ECN field and then is followed by another Context ID.

# Conventions

{::boilerplate bcp14}

# ECN enabled UDP Proxying Payload

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
  Reserved(6),
  ECN(2),
  UDP Proxying Payload (..) # Payload per another Context ID definition
}
~~~
{: #UDP-Proxying-Payload title="ECN carrying UDP Proxying Payload"}

Reserved: six bits currently reserved, SHALL be set to zero (0) on
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


# Negotiating ECN Usage


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

# Tunnel and ECN marking interactions

## Tunnel Endpoint Marking

The Tunnel Endpoint when receiving an IP/UDP packet that is belonging
to an Connect-UDP request where the ECN extension is enabled copies
the the two ECN field bits from the IP header to the ECN field in the
HTTP datagram payload using an Context ID that has the ECN field.

A Tunnel endpoint on egress copies the ECN extension field value into the
IP/UDP packet it creates for this UDP Proxying payload.

An Tunnel endpoint which is unable to read or set the ECN Field SHALL NOT
enable the ECN extension.


## Tunnel Transport Connection Interactions

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

As there would be no on the wire overhead to enable this field to
carry the DSCP one can ask if there is a point to include this. It
would however require more work to define.

Do we need to define a Context ID negotiation Capsule so that if another
extension in the future defines a capsule to define a context ID one
can combine it with ECN?

# IANA Considerations

IANA is request to register one new permanent Field name in the
Hypertext Transfer Protocol (HTTP) Field Name Registry (At time of
writing residing at:
https://www.iana.org/assignments/http-fields/http-fields.xhtml).

   Field Name: ECN-Context-ID

   Status: Permanent

   Structured Type: List

   Reference: [RFC-TO-BE]




# Acknowledgments


--- back
