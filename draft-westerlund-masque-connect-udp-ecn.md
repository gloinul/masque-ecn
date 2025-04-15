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
   RFC3168:
   RFC9298:
   RFC9297:
   RFC9330:


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


# Context


# Negotiating ECN Usage





# Acknowledgments


--- back
