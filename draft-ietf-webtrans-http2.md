---
title: "WebTransport using HTTP/2"
abbrev: "WebTransport-H2"
docname: draft-ietf-webtrans-http2-latest

date: {DATE}
category: std

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
    ins: M. Thomson
    name: Martin Thomson
    org: Mozilla
    email: mt@lowentropy.net
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

normative:
  OVERVIEW: I-D.ietf-webtrans-overview
  WEBTRANSPORT-H3: I-D.ietf-webtrans-http3
  HTTP: I-D.ietf-httpbis-semantics

informative:

--- abstract

WebTransport [OVERVIEW] is a protocol framework that enables clients constrained
by the Web security model to communicate with a remote server using a secure
multiplexed transport. This document describes a WebTransport protocol that
uses HTTP semantics to provide support for unidirectional streams,
bidirectional streams, and datagrams. This protocol can be used with any HTTP
version, however this document focuses on TCP-based HTTP versions, specifically
providing examples of these capabilities when multiplexed within the same
HTTP/2 [RFC7540] connection.

--- note_Note_to_Readers

Discussion of this draft takes place on the WebTransport mailing list
([webtransport@ietf.org](mailto:webtransport@ietf.org)), which is archived at
[](https://mailarchive.ietf.org/arch/search/?email_list=webtransport).

The repository tracking the issues for this draft can be found at
[](https://github.com/ietf-wg-webtrans/draft-webtransport-http2). The web API
draft corresponding to this document can be found at
[](https://w3c.github.io/webtransport/).

--- middle

# Introduction

This protocol satisfies the WebTransport protocol framework's requirements using
HTTP semantics to provide support for unidirectional streams, bidirectional
streams, and datagrams.

By using HTTP semantics, this protocol allows deployment using any HTTP version,
although this document in particular focuses on HTTP/2 as the current most
common TCP-based fallback to HTTP/3 running over QUIC {{?RFC9000}}.

Currently, the only mechanism in HTTP/2 for server to client communication is
server push. That is, servers can initiate unidirectional push promised streams
to clients, but clients cannot respond to them; they can only accept them or
discard them. Additionally, intermediaries along the path may have different
server push policies and may not forward push promised streams to the downstream
client. This best effort mechanism is not sufficient to reliably deliver
messages from servers to clients, limiting server to client use-cases such as
chat messages or notifications.

Several techniques have been developed to workaround these limitations: long
polling {{?RFC6202}}, WebSocket {{?RFC8441}}, and tunneling using the CONNECT
method. All of these approaches have limitations.

This document defines a mechanism for establishing bidirectional communication
using HTTP semantics in a manner that conforms with the WebTransport protocol
requirements and semantics [OVERVIEW]. When running with HTTP/2, multiple
WebTransport instances can be multiplexed simultaneously with regular HTTP
traffic on the same HTTP/2 connection.

It is important to note that, while it is possible to have the WebTransport
session remain entirely "self contained" within a given HTTP flow, certain HTTP
versions may provide native features, such as datagram support, which can allow
for a better performing implementation of the WebTransport client's
requirements.

## Terminology

The keywords "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{!RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

This document follows terminology defined in {{Section 1.2 of OVERVIEW}}. Note
that this document distinguishes between a WebTransport server and an HTTP/2
server. An HTTP/2 server is the server that terminates HTTP/2 connections; a
WebTransport server is an application that accepts WebTransport sessions, which
can be accessed via an HTTP/2 server.

# Protocol Overview

WebTransport servers are identified by an HTTPS URI as defined in {{Section
4.2.2 of HTTP}}.

When an HTTP/2 connection is established, both the client and server have to
send a SETTINGS_ENABLE_WEBTRANSPORT setting in order to indicate that they
both support WebTransport over HTTP/2.

WebTransport sessions are initiated inside a given HTTP/2 connection by the
client, who sends an extended CONNECT request {{!RFC8441}}. If the server
accepts the request, an WebTransport session is established. The resulting
stream will be further referred to as a *CONNECT stream*, and its stream ID is
used to uniquely identify a given WebTransport session within the connection.
The ID of the CONNECT stream that established a given WebTransport session will
be further referred to as a *Session ID*.

After the session is established, the peers can utilitze the bidirectional
bytestream established by the extended CONNECT request to exchange WebTransport
frames that multiplex flows of WebTransport data as *WebTransport streams*.
These frames closely mirror a subset of QUIC frames and provide the essential
WebTransport features, such as:

* Both client and server can create a bidirectional or unidirectional
  WebTransport stream within the WebTransport session using a WT_STREAM frame.
* A datagram can be sent using a WT_DATAGRAM frame.

Data flow on these streams is controlled via a flow control mechanism very
similar to the one provided by QUIC using WT_MAX_DATA, WT_MAX_STREAM_DATA,
WT_MAX_STREAMS, WT_DATA_BLOCKED, WT_STREAM_DATA_BLOCKED, and WT_STREAMS_BLOCKED
frames.

WebTransport streams can be closed via a WT_RESET_STREAM frame and a receiver
can request that a sender stop sending with a WT_STOP_SENDING frame.

A WebTransport session is terminated when the CONNECT stream that created it is
closed. This implicitly closes all WebTransport streams that were
multiplexed over that CONNECT stream.

# Session Establishment

## Establishing a Transport-Capable HTTP/2 Connection

In order to indicate support for WebTransport, both the client and the server
MUST send a SETTINGS_ENABLE_WEBTRANSPORT value set to "1" in their SETTINGS
frame. Endpoints MUST NOT use any WebTransport-related functionality unless the
parameter has been negotiated.

## Extended CONNECT in HTTP/2

{{!RFC8441}} defines an extended CONNECT method in {{features}}, enabled by the
SETTINGS_ENABLE_CONNECT_PROTOCOL parameter. An endpoint does not need to send
both SETTINGS_ENABLE_CONNECT_PROTOCOL and SETTINGS_ENABLE_WEBTRANSPORT; the
SETTINGS_ENABLE_WEBTRANSPORT setting implies that an endpoint supports extended
CONNECT.

## Creating a New Session

As WebTransport sessions are established over HTTP, they are identified
using the `https` URI scheme {{!RFC7230}}.

In order to create a new WebTransport session, a client can send an HTTP CONNECT
request. The `:protocol` pseudo-header field ({{!RFC8441}}) MUST be set to
`webtransport` ({{Section 7.1 of WEBTRANSPORT-H3}}). The `:scheme` field MUST be
`https`. Both the `:authority` and the `:path` value MUST be set; those fields
indicate the desired WebTransport server. In a Web context, the request MUST
include an `Origin` header field {{!ORIGIN=RFC6454}} that includes the origin of
the site that requested the creation of the session.

Upon receiving an extended CONNECT request with a `:protocol` field set to
`webtransport`, the HTTP server can check if it has a WebTransport server
associated with the specified `:authority` and `:path` values. If it does not,
it SHOULD reply with status code 404 (Section 6.5.4,{{!RFC7231}}). If it does,
it MAY accept the session by replying with status code 200. The WebTransport
server MUST verify the `Origin` header to ensure that the specified origin is
allowed to access the server in question.

From the client's perspective, a WebTransport session is established when the
client receives a 200 response. From the server's perspective, a session is
established once it sends a 200 response. Both endpoints MUST NOT send any
WebTransport frames on a given session before that session is established.

## Limiting the Number of Simultaneous Sessions

From a flow control perspective, WebTransport sessions count against HTTP/2
session flow control limits just like regular HTTP requests, since they are
established via an HTTP CONNECT request. This document does not make any effort
to introduce a separate flow control mechanism for WebTransport sessions. If
the server needs to limit the rate of incoming requests, it has alternative
mechanisms at its disposal:

* `HTTP_STREAM_REFUSED` error code defined in [RFC7540] indicates to the
  receiving HTTP/2 stack that the request was not processed in any way.
* HTTP status code 429 indicates that the request was rejected due to rate
  limiting {{!RFC6585}}. Unlike the previous method, this signal is directly
  propagated to the application.

# WebTransport Features {#features}

WebTransport over TCP-based HTTP semantics provides the following features
described in [OVERVIEW]: unidirectional streams, bidirectional streams, and
datagrams, initiated by either endpoint.

WebTransport streams and datagrams that belong to different WebTransport
sessions are identified by the CONNECT stream on which they are transmitted,
with one WebTransport session consuming one CONNECT stream.

## Transport Considerations

Because WebTransport over TCP-based HTTP semantics relies on the underlying
protocols to provide in order and reliable delivery, there are some notable
differences from the way in which QUIC handles application data.

Endpoints MUST send stream data in order. As there is no ordering mechanism
available for the receiver to reassemble incoming data, receivers assume that
all data arriving in STREAM frames is contiguous and in order.

DATAGRAM frames are delivered to the remote WebTransport endpoint reliably,
however this does not require that the receiving implementation deliver that
data to the application in a reliable manner.

## WebTransport Stream States

WebTransport streams have states that mirror the states of QUIC streams
({{Section 3 of !RFC9000}}) as closely as possible to aid in ease of
implementation.

Because WebTransport does not provide an acknowledgement mechanism for
WebTransport frames, it relies on the underlying transport's in order delivery
to inform stream state transitions. Wherever QUIC relies on receiving an ack
for a packet to transition between stream states, WebTransport performs that
transition immediately.

# WebTransport Frames

WebTransport frames mirror their QUIC counterparts as closely as possible to
enable maximal reuse of any applicable QUIC infrastructure by implementors.

A WebTransport frame begins with a Frame Type and Frame Length which are
followed by zero or more fields that are type-dependent.

~~~
Frame {
  Frame Type (i),
  Frame Length (i),
  Type-Dependent Fields (..),
}
~~~
{: #fig-wt_frame_header title="WebTransport Frame Format"}

The Frame Type field indicates the type of the frame, defining what
type-dependent fields will be present.

The Frame Length field indicates the length of the WebTransport frame, including
all type-dependent fields and other information. It does not include the size
of the Frame Type or Frame Length fields themselves.

Both of these fields use a variable-length integer encoding (see {{Section 16 of
!RFC9000}}), with one exception. To ensure simple and efficient
implementations of frame parsing, the frame type and length MUST use the
shortest possible encoding. For example, for the frame types defined in this
document, this means a single-byte encoding, even though it is possible to
encode these values as a two-, four-, or eight-byte variable-length integer.

## WT_PADDING Frames

A WT_PADDING frame (type=0x00) has no semantic value. PADDING frames can be used
to introduce additional data between other WebTransport frames and can also be
used to provide protection against traffic analysis or for other reasons.

~~~
WT_PADDING Frame {
  Type (i) = 0x00,
  Length (i),
  Padding (..),
}
~~~
{: #fig-wt_padding title="WT_PADDING Frame Format"}

The Padding field MUST be set to an all-zero sequence of bytes of any length as
specified by the Length field.
<!-- TODO validation and error handling -->

## WT_RESET_STREAM Frames

A WebTransport frame called WT_RESET_STREAM is introduced for either endpoint to
abruptly terminate the sending part of a WebTransport stream.

An endpoint uses a WT_RESET_STREAM frame (type=0x04) to abruptly terminate the
sending part of a stream.

After sending a WT_RESET_STREAM, an endpoint ceases transmission and
retransmission of WT_STREAM frames on the identified stream. A receiver of
WT_RESET_STREAM can discard any data that it already received on that stream.

~~~
WT_RESET_STREAM Frame {
  Type (i) = 0x04,
  Length (i) = 0x00,
  Stream ID (i),
  Application Protocol Error Code (i),
  Final Size (i),
}
~~~
{: #fig-wt_reset_stream title="WT_RESET_STREAM Frame Format"}

The WT_RESET_STREAM frame defines the following fields:

   Stream ID: A variable-length integer encoding of the WebTransport stream ID
   of the stream being terminated.

   Application Protocol Error Code: A variable-length integer containing the
   application protocol error code that indicates why the stream is being
   closed.

   Final Size: A variable-length integer indicating the final size of the stream
   by the WT_RESET_STREAM sender, in units of bytes. This is the amount of flow
   control credit that is consumed by a stream, see {{Section 4.5 of !RFC9000}}.

## WT_STOP_SENDING Frames

A WebTransport frame called WT_STOP_SENDING is introduced to communicate that
incoming data is being discarded on receipt per application request.
WT_STOP_SENDING requests that a peer cease transmission on a stream.

~~~
WT_STOP_SENDING Frame {
  Type (i) = 0x05,
  Length (i) = 0x00,
  Stream ID (i),
  Application Protocol Error Code (i),
}
~~~
{: #fig-wt_stop_sending title="WT_STOP_SENDING Frame Format"}

The WT_STOP_SENDING frame defines the following fields:

   Stream ID: A variable-length integer carrying the WebTransport stream ID of
   the stream being ignored.

   Application Protocol Error Code: A variable-length integer containing the
   application-specified reason the sender is ignoring the stream.


## WT_STREAM Frames

WT_STREAM frames implicitly create a stream and carry stream data. The Type
field in the WT_STREAM frame takes the form 0b00001XXX (or the set of values
from 0x08 to 0x0f) to maximize compatibility with QUIC. However, unlike QUIC,
there are only one bit used to determine the fields that are present in the
frame:

The FIN bit (0x01) indicates that the frame marks the end of the stream. The
final size of the stream is the sum of the length, in bytes, of all data
previously sent on this stream and the length of this frame without the length
of the Stream ID.

~~~
WT_STREAM Frame {
  Type (i) = 0x08..0x0f,
  Stream ID (i),
  Stream Data (..),
}
~~~
{: #fig-wt_stream title="WT_STREAM Frame Format"}

## WT_MAX_DATA Frames

A WebTransport frame called WT_MAX_DATA is introduced to inform the peer of the
maximum amount of data that can be sent on the WebTransport session as a
whole.

~~~
WT_MAX_DATA Frame {
  Type (i) = 0x10,
  Length (i) = 0x00,
  Maximum Data (i),
}
~~~
{: #fig-wt_max_data title="WT_MAX_DATA Frame Format"}

WT_MAX_DATA frames contain the following field:

   Maximum Data: A variable-length integer indicating the maximum amount of data
   that can be sent on the entire connection, in units of bytes.

All data sent in WT_STREAM frames counts toward this limit. The sum of the
lengths of Stream Data fields in WT_STREAM frames MUST NOT exceed the value
advertised by a receiver.

## WT_MAX_STREAM_DATA Frames

A WebTransport frame called WT_MAX_STREAM_DATA is introduced to inform a peer of
the maximum amount of data that can be sent on a stream.

~~~
WT_MAX_STREAM_DATA Frame {
  Type (i) = 0x11,
  Length (i) = 0x00,
  Stream ID (i),
  Maximum Stream Data (i),
}
~~~
{: #fig-wt_max_stream_data title="WT_MAX_STREAM_DATA Frame Format"}

WT_MAX_STREAM_DATA frames contain the following fields:

   Stream ID: The stream ID of the affected WebTransport stream, encoded as a
   variable-length integer.

   Maximum Stream Data: A variable-length integer indicating the maximum amount
   of data that can be sent on the identified stream, in units of bytes.

All data sent in WT_STREAM frames for the identified stream counts toward this
limit. The sum of the lengths of Stream Data fields in WT_STREAM frames on the
identified stream MUST NOT exceed the value advertised by a receiver.

## WT_MAX_STREAMS Frames

A WebTransport frame called WT_MAX_STREAMS is introduced to inform the peer of
the cumulative number of streams of a given type it is permitted to open. A
WT_MAX_STREAMS frame with a type of 0x12 applies to bidirectional streams, and
a WT_MAX_STREAMS frame with a type of 0x13 applies to unidirectional streams.

~~~
WT_MAX_STREAMS Frame {
  Type (i) = 0x12..0x13,
  Length (i) = 0x00,
  Maximum Streams (i),
}
~~~
{: #fig-wt_max_streams title="WT_MAX_STREAMS Frame Format"}

WT_MAX_STREAMS frames contain the following field:

   Maximum Streams: A count of the cumulative number of streams of the
   corresponding type that can be opened over the lifetime of the connection.
   This value cannot exceed 2^60, as it is not possible to encode stream IDs
   larger than 2^(62-1).

An endpoint MUST NOT open more streams than permitted by the current stream
limit set by its peer. For instance, a server that receives a unidirectional
stream limit of 3 is permitted to open streams 3, 7, and 11, but not stream 15.

Note that this limit includes streams that have been closed as well as those
that are open.

## WT_DATA_BLOCKED Frames

A sender SHOULD send a WT_DATA_BLOCKED frame (type=0x14) when it wishes to send
data but is unable to do so due to WebTransport session-level flow control.
WT_DATA_BLOCKED frames can be used as input to tuning of flow control
algorithms.

~~~
WT_DATA_BLOCKED Frame {
  Type (i) = 0x14,
  Length (i) = 0x00,
  Maximum Data (i),
}
~~~
{: #fig-wt_data_blocked title="WT_DATA_BLOCKED Frame Format"}

WT_DATA_BLOCKED frames contain the following field:

   Maximum Data: A variable-length integer indicating the session-level limit
   at which blocking occurred.

## WT_STREAM_DATA_BLOCKED Frames

A sender SHOULD send a WT_STREAM_DATA_BLOCKED frame (type=0x15) when it wishes
to send data but is unable to do so due to stream-level flow control. This
frame is analogous to WT_DATA_BLOCKED.

~~~
WT_STREAM_DATA_BLOCKED Frame {
  Type (i) = 0x15,
  Length (i) = 0x00,
  Stream ID (i),
  Maximum Stream Data (i),
}
~~~
{: #fig-wt_stream_data_blocked title="WT_STREAM_DATA_BLOCKED Frame Format"}

WT_STREAM_DATA_BLOCKED frames contain the following fields:

   Stream ID: A variable-length integer indicating the WebTransport stream that
   is blocked due to flow control.

   Maximum Stream Data: A variable-length integer indicating the offset of the
   stream at which the blocking occurred.

## WT_STREAMS_BLOCKED Frames

A sender SHOULD send a WT_STREAMS_BLOCKED frame (type=0x16 or 0x17) when it
wishes to open a stream but is unable to do so due to the maximum stream limit
set by its peer. A WT_STREAMS_BLOCKED frame of type 0x16 is used to indicate
reaching the bidirectional stream limit, and a STREAMS_BLOCKED frame of type
0x17 is used to indicate reaching the unidirectional stream limit.

A WT_STREAMS_BLOCKED frame does not open the stream, but informs the peer that a
new stream was needed and the stream limit prevented the creation of the
stream.

~~~
WT_STREAMS_BLOCKED Frame {
  Type (i) = 0x16..0x17,
  Length (i) = 0x00,
  Maximum Streams (i),
}
~~~
{: #fig-wt_streams_blocked title="WT_STREAMS_BLOCKED Frame Format"}

WT_STREAMS_BLOCKED frames contain the following field:

   Maximum Streams: A variable-length integer indicating the maximum number of
   streams allowed at the time the frame was sent. This value cannot exceed
   2^60, as it is not possible to encode stream IDs larger than 2^(62-1).

## WT_DATAGRAM Frames

The WT_DATAGRAM frame type (0x31) is used to carry datagram traffic. Frame type
0x30 is also reserved to maintain parity with QUIC, but unused, as all
WebTransport frames MUST contain a length field.

~~~
WT_DATAGRAM Frame {
  Type (i) = 0x31,
  Length (i),
  Datagram Data (..),
}
~~~
{: #fig-wt_datagram title="WT_DATAGRAM Frame Format"}

WT_DATAGRAM frames contain the following fields:

   Length: A variable-length integer specifying the length of the Datagram Data
   in bytes. Note that empty (i.e., zero-length) datagrams are allowed.

   Datagram Data: The content of the datagram to be delivered.

The data in WT_DATAGRAM frames is not subject to flow control. The receiver MAY
discard this data if it does not have sufficient space to buffer it.

An intermediary could forward the data in a WT_DATAGRAM frame over another
protocol, such as WebTransport over HTTP/3. In QUIC, a datagram frame can span
at most one packet. Because of that, the applications have to know the maximum
size of the datagram they can send. However, when proxying the datagrams, the
hop-by-hop MTUs can vary.

# Examples

An example of negotiating a WebTransport Stream on an HTTP/2 connection follows.
This example is intended to closely follow the example in {{Section 5.1 of
!RFC8441}} to help illustrate the differences defined in this document.

~~~
[[ From Client ]]                   [[ From Server ]]

SETTINGS
SETTINGS_ENABLE_WEBTRANSPORT = 1

                                    SETTINGS
                                    SETTINGS_ENABLE_WEBTRANSPORT = 1

HEADERS + END_HEADERS
Stream ID = 3
:method = CONNECT
:protocol = webtransport
:scheme = https
:path = /
:authority = server.example.com
origin: server.example.com

                                    HEADERS + END_HEADERS
                                    Stream ID = 3
                                    :status = 200

WT_STREAM
Stream ID = 5
WebTransport Data

                                    WT_STREAM + FIN
                                    Stream ID = 5
                                    WebTransport Data

WT_STREAM + FIN
Stream ID = 5
WebTransport Data
~~~

An example of the server initiating a WebTransport Stream follows. The only
difference here is the endpoint that sends the first WT_STREAM frame.

~~~
[[ From Client ]]                   [[ From Server ]]

SETTINGS
SETTINGS_ENABLE_WEBTRANSPORT = 1

                                    SETTINGS
                                    SETTINGS_ENABLE_WEBTRANSPORT = 1

HEADERS + END_HEADERS
Stream ID = 3
:method = CONNECT
:protocol = webtransport
:scheme = https
:path = /
:authority = server.example.com
origin: server.example.com
                                    HEADERS + END_HEADERS
                                    Stream ID = 3
                                    :status = 200

                                    WT_STREAM
                                    Stream ID = 2
                                    WebTransport Data

WT_STREAM + FIN
Stream ID = 2
WebTransport Data

                                    WT_STREAM + FIN
                                    Stream ID = 2
                                    WebTransport Data
~~~


# Session Termination

An WebTransport session over HTTP/2 is terminated when either endpoint closes
the stream associated with the CONNECT request that initiated the session.
Upon learning about the session being terminated, the endpoint MUST stop
sending new datagrams and reset all of the streams associated with the session.

# Transport Properties

The WebTransport framework [OVERVIEW] defines a set of optional transport
properties that clients can use to determine the presence of features which
might allow additional optimizations beyond the common set of properties
available via all WebTransport protocols. Below are details about support in
Http2Transport for those properties.

Stream Independence:

: Http2Transport does not support stream independence, as HTTP/2 inherently has
    head of line blocking.

Partial Reliability:

: Http2Transport does not support partial reliability, as HTTP/2 retransmits any
  lost data. This means that any datagrams sent via Http2Transport will be
  retransmitted regardless of the preference of the application. The receiver
  is permitted to drop them, however, if it is unable to buffer them.

Pooling Support:

: Http2Transport supports pooling, as multiple transports using Http2Transport
    may share the same underlying HTTP/2 connection and therefore share a
    congestion controller and other transport context.

Connection Mobility:

: Http2Transport does not support connection mobility, unless an underlying
    transport protocol that supports multipath or migration, such as MPTCP
    {{?RFC7540}}, is used underneath HTTP/2 and TLS. Without such support,
    Http2Transport connections cannot survive network transitions.

# Security Considerations

WebTransport over HTTP/2 satisfies all of the security requirements imposed by
[OVERVIEW] on WebTransport protocols, thus providing a secure framework for
client-server communication in cases when the client is potentially untrusted.

WebTransport over HTTP/2 requires explicit opt-in through the use of HTTP
SETTINGS; this avoids potential protocol confusion attacks by ensuring the
HTTP/2 server explicitly supports it. It also requires the use of the Origin
header, providing the server with the ability to deny access to Web-based
clients that do not originate from a trusted origin.

Just like HTTP traffic going over HTTP/2, WebTransport pools traffic to
different origins within a single connection. Different origins imply different
trust domains, meaning that the implementations have to treat each transport as
potentially hostile towards others on the same connection. One potential attack
is a resource exhaustion attack: since all of the transports share both
congestion control and flow control context, a single client aggressively using
up those resources can cause other transports to stall. The user agent thus
SHOULD implement a fairness scheme that ensures that each transport within
connection gets a reasonable share of controlled resources; this applies both
to sending data and to opening new streams.

# IANA Considerations

## HTTP/2 SETTINGS Parameter Registration

The following entry is added to the "HTTP/2 Settings" registry established by
[RFC7540]:

The `SETTINGS_ENABLE_WEBTRANSPORT` parameter indicates that the specified
HTTP/2 connection is WebTransport-capable.

Setting Name:

: ENABLE_WEBTRANSPORT

Value:

: 0x2b603742

Default:

: 0

Specification:

: This document


--- back

# Acknowledgments
{:numbered="false"}

Thanks to Anthony Chivetta, Joshua Otto, and Valentin Pistol for their
contributions in the design and implementation of this work.
