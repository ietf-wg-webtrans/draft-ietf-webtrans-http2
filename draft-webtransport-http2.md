---
title: "WebTransport using HTTP/2"
abbrev: "WebTransportHTTP2"
docname: draft-webtransport-http2-latest

date: {DATE}
category: std
consensus: true

ipr: trust200902
area: art
workgroup: webtrans
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: A. Frindell
    name: Alan Frindell
    organization: Facebook Inc.
    email: afrind@fb.com
 -
    ins: E. Kinnear
    name: Eric Kinnear
    org: Apple Inc.
    street: One Apple Park Way
    city: Cupertino, California 95014
    country: United States of America
    email: ekinnear@apple.com
 -
    ins: T. Pauly
    name: Tommy Pauly
    org: Apple Inc.
    street: One Apple Park Way
    city: Cupertino, California 95014
    country: United States of America
    email: tpauly@apple.com
 -
    ins: V. Vasiliev
    name: Victor Vasiliev
    organization: Google
    email: vasilvv@google.com
 -
    ins: G. Xie
    name: Guowu Xie
    organization: Facebook Inc.
    email: woo@fb.com

--- abstract

WebTransport is a protocol framework that enables clients constrained by the Web
security model to communicate with a remote server using a secure multiplexed
transport. This document describes Http2Transport, a WebTransport protocol that
is based on HTTP/2 and provides support for bidirectional streams multiplexed
within the same HTTP/2 connection.

--- note_Note_to_Readers

Discussion of this draft takes place on the WebTransport mailing list
(webtransport@ietf.org), which is archived at
\<https://mailarchive.ietf.org/arch/search/?email_list=webtransport\>.

The repository tracking the issues for this draft can be found at
\<https://github.com/erickinnear/draft-http-transport/issues\>.
The web API draft corresponding to this document can be found at
\<https://wicg.github.io/web-transport/\>.

--- middle

# Introduction

HTTP/2 {{?RFC7540}} transports HTTP messages via a framing layer that includes
many technologies and optimizations designed to make communication more
efficient between clients and servers. These include multiplexing of multiple
streams on a single underlying transport connection, flow control, stream
dependencies and priorities, header compression, and exchange of configuration
information between endpoints.

Currently, the only mechanism in HTTP/2 for server to client communication is
server push. That is, servers can initiate unidirectional push promised streams
to clients, but clients cannot respond to them and either accept or discard them
silently. Additionally, intermediaries along the path may have different server
push policies and may not forward push promised streams to the downstream
client. This best effort mechanism is not sufficient to reliably deliver content
from servers to clients, limiting additional use-cases such as sending messages
and notifications from servers to clients immediately when they become
available.

Several techniques have been developed to workaround these limitations: long
polling {{?RFC6202}}, WebSocket {{?RFC8441}}, and tunneling using the CONNECT
method. All of these approaches layer an application protocol on top of HTTP/2,
using HTTP/2 streams as transport connections. This layering defeats the
optimizations provided by HTTP/2. For example, multiplexing multiple parallel
interactions onto one HTTP/2 stream reintroduces head of line blocking. Also,
application metadata is encapsulated into DATA frames, rather than HEADERS
frames, making header compression impossible. Further, user data is framed
multiple times at different protocol layers, which offsets the wire efficiency
of HTTP/2 binary framing.

This document defines Http2Transport, a mechanism for multiplexing non-HTTP data
with HTTP/2 in a manner that conforms with the WebTransport protocol framework
{{?I-D.vvv-webtransport-overview}}. Using the mechanism described, multiple
Http2Transport instances can be multiplexed simultaneously with regular HTTP
traffic on the same HTTP/2 connection.

Section 8.3 of {{?RFC7540}} defines the HTTP CONNECT method for HTTP/2, which
converts a HTTP/2 stream into a tunnel for arbitrary data. {{?RFC8441}}
describes the use of the extended CONNECT method to negotiate the use of the
WebSocket Protocol {{?RFC6455}} on an HTTP/2 stream. Http2Transport uses the
extended CONNECT handshake to allow WebTransport endpoints to multiplex
arbitrary data on HTTP/2 streams.

In this draft, a new HTTP/2 frame is introduced which has the routing properties
of a PUSH_PROMISE frame and the bi-directionality of a HEADERS frame. The
extension provides several benefits:

1. After a HTTP/2 connection is established, a server can initiate streams to
the client at any time, and the client can respond to the incoming streams
accordingly. That is, the communication over HTTP/2 is bidirectional and
symmetric.

2. All of the HTTP/2 technologies and optimizations still apply. Intermediaries
also have all the necessary metadata to properly handle the communication
between the client and the server.

3. Clients are able to group streams together for routing purposes, such that
each individual stream group can be used for a different service, within the
same HTTP/2 connection.


## Terminology

This document follows terminology defined in Section 1.2 of
{{?I-D.vvv-webtransport-overview}}. Note that this document distinguishes
between a WebTransport server and an HTTP/2 server. An HTTP/2 server is the
server that terminates HTTP/2 connections; a WebTransport server is an
application that accepts WebTransport sessions, which can be accessed via an
HTTP/2 server.


# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{!RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.


# Security Considerations

Security


# IANA Considerations

This document has no IANA actions.



--- back

# Acknowledgments
{:numbered="false"}

Acknowledge.


