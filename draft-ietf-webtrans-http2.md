---
title: "WebTransport over HTTP/2"
abbrev: "WebTransport-H2"
docname: draft-ietf-webtrans-http2-latest
number:
date:
consensus: true
v: 3
category: std
area: "Web and Internet Transport"
wg: WEBTRANS
venue:
  group: "WebTransport"
  type: "Working Group"
  mail: "webtransport@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/webtransport/"
  github: "ietf-wg-webtrans/draft-ietf-webtrans-http2"
  latest: "https://ietf-wg-webtrans.github.io/draft-ietf-webtrans-http2/#go.draft-ietf-webtrans-http2.html"
keyword:
  - webtransport
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
  HTTP-DATAGRAM: RFC9297
  HTTP2: RFC9113

informative:
  DATAGRAM: RFC9221

--- abstract

WebTransport defines a set of low-level communications features designed for
client-server interactions that are initiated by Web clients.  This document
describes a protocol that can provide many of the capabilities of WebTransport
over HTTP/2.  This protocol enables the use of WebTransport when a UDP-based
protocol is not available.

--- middle

# Introduction

WebTransport {{OVERVIEW}} is designed to provide generic communication
capabilities to Web clients that use HTTP/3 {{?HTTP3=I-D.ietf-quic-http}}.  The
HTTP/3 WebTransport protocol {{WEBTRANSPORT-H3}} allows Web clients to use QUIC
{{?QUIC=RFC9000}} features such as streams or datagrams {{DATAGRAM}}.
However, there are some environments where QUIC cannot be deployed.

This document defines a protocol that provides all of the core functions of
WebTransport using HTTP semantics. This includes unidirectional streams,
bidirectional streams, and datagrams.

By relying only on generic HTTP semantics, this protocol might allow deployment
using any HTTP version.  However, this document only defines negotiation for
HTTP/2 {{HTTP2}} as the current most common TCP-based fallback to HTTP/3.

## Terminology

The keywords "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{!RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

This document follows terminology defined in {{Section 1.2 of OVERVIEW}}. Note
that this document distinguishes between a WebTransport server and an HTTP/2
server. An HTTP/2 server is the server that terminates HTTP/2 connections; a
WebTransport server is an application that accepts WebTransport sessions, which
can be accessed using HTTP/2 and this protocol.

# Protocol Overview

WebTransport servers are identified by an HTTPS URI as defined in {{Section
4.2.2 of HTTP}}.

A client initiates a WebTransport session by sending an extended CONNECT request
{{!RFC8441}}. If the server accepts the request, a WebTransport session is
established. The stream that carries the CONNECT request is used to exchange
bidirectional data for the session. This stream will be referred to as a
*CONNECT stream*.  The stream ID of a CONNECT stream, which will be referred to
as a *Session ID*, is used to uniquely identify a given WebTransport session
within the connection.  WebTransport using HTTP/2 uses extended CONNECT with
the same `webtransport` HTTP Upgrade Token as {{WEBTRANSPORT-H3}}.  This
Upgrade Token uses the Capsule Protocol as defined in {{HTTP-DATAGRAM}}.

After the session is established, endpoints exchange WebTransport messages using
the Capsule Protocol on the bidirectional CONNECT stream, the "data stream" as
defined in {{Section 3.1 of HTTP-DATAGRAM}}.

Within this stream, *WebTransport streams* and *WebTransport datagrams* are
multiplexed.  In HTTP/2, WebTransport capsules are carried in HTTP/2 DATA
frames. Multiple independent WebTransport sessions can share a connection if
the HTTP version supports that, as HTTP/2 does.

WebTransport capsules closely mirror a subset of QUIC frames and provide the
essential WebTransport features.  Within a WebTransport session, endpoints can
create and use bidirectional or unidirectional streams with no additional round
trips using the WT_STREAM capsule.

Stream creation and data flow on streams uses flow control mechanisms modeled on
those in QUIC. Flow control is managed using the WebTransport capsules:
WT_MAX_DATA, WT_MAX_STREAM_DATA, WT_MAX_STREAMS, WT_DATA_BLOCKED,
WT_STREAM_DATA_BLOCKED, and WT_STREAMS_BLOCKED. Flow control for the CONNECT
stream as a whole, as provided by the HTTP version in use, applies in addition
to any WebTransport-session-level flow control.

WebTransport streams can be aborted using a WT_RESET_STREAM capsule and a
receiver can request that a sender stop sending with a WT_STOP_SENDING
capsule.

A WebTransport session is terminated when the CONNECT stream that created it is
closed. This implicitly closes all WebTransport streams that were
multiplexed over that CONNECT stream.

# Session Establishment and Termination

A WebTransport session is a communication context between a client and server
{{OVERVIEW}}. This section describes how sessions begin and end.

## Establishing a WebTransport-Capable HTTP/2 Connection

In order to indicate potential support for WebTransport, the server MUST send the
SETTINGS_ENABLE_CONNECT_PROTOCOL setting with a value of "1" in its SETTINGS
frame.  The client MUST NOT send a WebTransport request until it has received
the setting indicating extended CONNECT support from the server.

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
`webtransport`, the HTTP server checks if the identified resource supports
WebTransport sessions. If the resource does not, the server SHOULD reply with
status code 406 ({{Section 15.5.7 of !RFC9110}}). If it does, it MAY accept the
session by replying with a 2xx series status code, as defined in {{Section 15.3
of !SEMANTICS=I-D.ietf-httpbis-semantics}}. The WebTransport server MUST verify
the `Origin` header to ensure that the specified origin is allowed to access
the server in question.

A WebTransport session is established when the server sends a 2xx response. A
server generates that response from the request header, not from the contents
of the request. To enable clients to resend data when attempting to
re-establish a session that was rejected by a server, a server MUST NOT process
any capsules on the request stream unless it accepts the WebTransport session.

A client MAY optimistically send any WebTransport capsules associated with a
CONNECT request, without waiting for a response, to the extent allowed by flow
control. This can reduce latency for data sent by a client at the start of a
WebTransport session. For example, a client might choose to send datagrams or
flow control updates before receiving any response from the server.

## Application Protocol Negotiation

WebTransport over HTTP/2 offers a subprotocol negotiation mechanism, similar to
TLS Application-Layer Protocol Negotiation Extension (ALPN) {{?RFC7301}}; the
intent is to simplify porting pre-existing protocols that rely on this type of
functionality.

The user agent MAY include a `WT-Available-Protocols` header field in the
CONNECT request. The `WT-Available-Protocols` enumerates the possible protocols
in preference order. If the server receives such a header, it MAY include a
`WT-Protocol` field in a successful (2xx) response. If it does, the server
MUST include a single choice from the client's list in that field. Servers MAY
reject the request if the client did not include a suitable protocol.

Both `WT-Available-Protocols` and `WT-Protocol` are defined in {{Section 3.4
of WEBTRANSPORT-H3}}.

## Session Termination and Error Handling {#errors}

An WebTransport session over HTTP/2 is terminated when either endpoint closes
the stream associated with the CONNECT request that initiated the session.

Prior to closing the stream associated with the CONNECT request, either
endpoint can send a WT_CLOSE_SESSION capsule with an application error code
and message to convey additional information about the reasons for the
closure of the session.

Session errors result in the termination of a session.  Errors can be reported
using the WT_CLOSE_SESSION capsule, which includes an error code and an
optional explanatory message.

An endpoint can terminate a session without sending a WT_CLOSE_SESSION capsule
by closing the HTTP/2 stream.

An HTTP/2 stream that is reset terminates the session without providing an
application-level signal, though there will be an HTTP/2 error code.

This document reserves the following HTTP/2 error codes for use with reporting
WebTransport errors:

WEBTRANSPORT_ERROR (0xTBD):
: This generic error can be used for errors that do not have more specific error
  codes.

WEBTRANSPORT_STREAM_STATE_ERROR (0xTBD):
: A stream-related capsule identified a stream that was in an invalid state.

Prior to terminating a stream with an error, a WT_CLOSE_SESSION capsule with an
application-specified error code MAY be sent.

Session errors do not necessarily result in any change of HTTP/2 connection
state, except that an endpoint might choose to terminate a connection in
response to stream errors; see {{Section 5.4 of HTTP2}}.

# Flow Control

Flow control governs the amount of resources that can be consumed or data that
can be sent.  WebTransport over HTTP/2 allows a server to limit the number of
sessions that a client can create on a single connection; see
{{flow-control-limit-sessions}}.

For data, there are five applicable levels of flow control for data that is sent
or received using WebTransport over HTTP/2:

1. TCP flow control.

2. HTTP/2 connection flow control, which governs the total amount of data in
DATA frames for all HTTP/2 streams.

3. HTTP/2 stream flow control, which limits the data on a single HTTP/2 stream.
For a WebTransport session, this includes all capsules, including those that
are exempt from WebTransport session-level flow control.

4. WebTransport session-level flow control, which limits the total amount of
stream data that can be sent or received on streams within the WebTransport
session. Note that this does not limit other types of capsules within a
WebTransport session, such as control messages or datagrams.

5. WebTransport stream flow control, which limits data on individual streams
within a session.

TCP and HTTP/2 define the first three levels of flow control. This document
defines the final two.

## Limiting the Number of Simultaneous Sessions {#flow-control-limit-sessions}

HTTP/2 defines a SETTINGS_MAX_CONCURRENT_STREAMS parameter {{Section 6.5.2 of
HTTP2}} that allows the server to limit the maximum number of concurrent streams
on a single HTTP/2 session, which also limits the number of WebTransport
sessions on that connection.  Servers that wish to limit the rate of incoming
WebTransport sessions on any particular HTTP/2 session have multiple mechanisms:

* The `REFUSED_STREAM` error code defined in {{Section 8.7 of HTTP2}}
  indicates to the receiving HTTP/2 stack that the request was not processed in
  any way.
* HTTP status code 429 indicates that the request was rejected due to rate
  limiting {{!RFC6585}}.  Unlike the previous method, this signal is directly
  propagated to the application.

## Limiting the Number of Streams Within a Session {#flow-control-limit-streams}

The WT_MAX_STREAMS capsule ({{Section 5.6.1 of WEBTRANSPORT-H3}}) allows
each endpoint to limit the number of streams its peer is permitted to open as
part of a WebTransport session.  There is a separate limit for bidirectional
streams and for unidirectional streams.  Note that, unlike WebTransport over
HTTP/3 {{WEBTRANSPORT-H3}}, because the entire WebTransport session is
contained within HTTP/2 DATA frames on a single HTTP/2 stream, this limit is
the only mechanism for an endpoint to limit the number of WebTransport streams
that its peer can open on a session.

## Initial Flow Control Limits {#flow-control-initial}

To allow stream data to be exchanged in the same flight as the extended CONNECT
request that establishes a WebTransport session, initial flow control limits
can be exchanged via HTTP/2 SETTINGS ({{flow-control-settings}}).  Initial
values for the flow control limits can also be exchanged via the
`WebTransport-Init` header field on the extended CONNECT request
({{flow-control-header}}).

The limits communicated via HTTP/2 SETTINGS apply to all WebTransport sessions
opened on that HTTP/2 connection.  Limits communicated via the
`WebTransport-Init` header field apply only to the WebTransport session
established by the extended CONNECT request carrying that field.

If both the SETTINGS and the header field are present when a WebTransport
session is established, the endpoint MUST use the greater of the two values for
each corresponding initial flow control value.  Endpoints sending the SETTINGS
and also including the header field SHOULD ensure that the header field values
are greater than or equal to the values provided in the SETTINGS.

### Flow Control SETTINGS {#flow-control-settings}

*[SETTINGS_WT_INITIAL_MAX_DATA]: #
*[SETTINGS_WT_INITIAL_MAX_STREAM_DATA_UNI]: #
*[SETTINGS_WT_INITIAL_MAX_STREAM_DATA_BIDI]: #
*[SETTINGS_WT_INITIAL_MAX_STREAMS_UNI]: #
*[SETTINGS_WT_INITIAL_MAX_STREAMS_BIDI]: #

Initial flow control limits can be exchanged via HTTP/2 SETTINGS
({{h2-settings}}) by providing non-zero values for

* WT_MAX_DATA via SETTINGS_WT_INITIAL_MAX_DATA
* WT_MAX_STREAM_DATA via SETTINGS_WT_INITIAL_MAX_STREAM_DATA_UNI and
  SETTINGS_WT_INITIAL_MAX_STREAM_DATA_BIDI
* WT_MAX_STREAMS via SETTINGS_WT_INITIAL_MAX_STREAMS_UNI and
  SETTINGS_WT_INITIAL_MAX_STREAMS_BIDI


### Flow Control Header Field {#flow-control-header}

The `WebTransport-Init` HTTP header field can be used to communicate the initial
values of the flow control windows, similar to how QUIC uses transport
parameters.  The `WebTransport-Init` is a Dictionary Structured Field ({{Section
3.2 of !RFC8941}}).  If the `WebTransport-Init` field cannot be parsed
correctly or does not have the correct type, the endpoint MUST reject the
CONNECT request with a 4xx status code.  The following keys are defined for the
`WebTransport-Init` header field:

`u`:
: The initial flow control limit for unidirectional streams opened by the
  recipient of this header field.  MUST be an Integer.

`bl`:
: The initial flow control limit for the bidirectional streams opened by the
  sender of this header field.  MUST be an Integer.

`br`:
: The initial flow control limit for the bidirectional streams opened by the
  recipient of this header field.  MUST be an Integer.

If any of these keys are present but contain invalid values, the endpoint MUST
reject the CONNECT request with a 4xx status code.  Unknown keys and parameters
in the dictionary MUST be ignored.

## Flow Control and Intermediaries {#flow-control-intermediaries}

WebTransport over HTTP/2 uses several capsules for flow control, and all of
these capsules define special intermediary handling as described in
{{Section 3.2 of HTTP-DATAGRAM}}.  These capsules, referred to as the "flow
control capsules" are WT_MAX_DATA, WT_MAX_STREAM_DATA, WT_MAX_STREAMS,
WT_DATA_BLOCKED, WT_STREAM_DATA_BLOCKED, and WT_STREAMS_BLOCKED.

Because flow control in WebTransport is hop-by-hop and does not provide an
end-to-end signal, intermediaries MUST consume flow control signals and express
their own flow control limits to the next hop. The intermediary can send these
signals via HTTP/3 flow control messages, HTTP/2 flow control messages, or as
WebTransport flow control capsules, where appropriate. Intermediaries are
responsible for storing any data for which they advertise flow control credit
if that data cannot be immediately forwarded to the next hop.

In practice, an intermediary that translates flow control signals between
similar WebTransport protocols, such as between two HTTP/2 connections, can
often simply reexpress the same limits received on one connection directly on
the other connection.

An intermediary that does not want to be responsible for storing data that
cannot be immediately sent on its translated connection would ensure that it
does not advertise a higher flow control limit on one connection than the
corresponding limit on the translated connection.

# WebTransport Features {#features}

WebTransport over TCP-based HTTP semantics provides the following features
described in [OVERVIEW]: unidirectional streams, bidirectional streams, and
datagrams, initiated by either endpoint.

WebTransport streams and datagrams that belong to different WebTransport
sessions are identified by the CONNECT stream on which they are transmitted,
with one WebTransport session consuming one CONNECT stream.

## Transport Properties

The WebTransport framework {{OVERVIEW}} defines a set of optional transport
properties that clients can use to determine the presence of features which
might allow additional optimizations beyond the common set of properties
available via all WebTransport protocols.

Because WebTransport over TCP-based HTTP semantics relies on the underlying
protocols to provide in order and reliable delivery, there are some notable
differences from the way in which QUIC handles application data. For example,
endpoints send stream data in order. As there is no ordering mechanism
available for the receiver to reassemble incoming data, receivers assume that
all data arriving in WT_STREAM capsules is contiguous and in order.

Below are details about support in WebTransport over HTTP/2 for the properties
defined by the WebTransport framework.

Unreliable Delivery:

: WebTransport over HTTP/2 does not support unreliable delivery, as HTTP/2
  retransmits any lost data. This means that any datagrams sent via
  WebTransport over HTTP/2 will be retransmitted regardless of the preference
  of the application. The receiver is permitted to drop them, however, if it is
  unable to buffer them.

Pooling:

: WebTransport over HTTP/2 provides support for pooling.  Every
  WebTransport session is an independent HTTP/2 stream and does not compete with
  other pooled WebTransport sessions for per-stream resources.

Stream Independence:

: WebTransport over HTTP/2 does not support stream independence, as HTTP/2
  inherently has head-of-line blocking.

Connection Mobility:

: WebTransport over HTTP/2 does not support connection mobility, unless an
  underlying transport protocol that supports multipath or migration, such as
  MPTCP {{?MPTCP=RFC6824}}, is used underneath HTTP/2 and TLS. Without such
  support, WebTransport over HTTP/2 connections cannot survive network
  transitions.

## WebTransport Streams

WebTransport streams have identifiers and states that mirror the identifiers
(({{Section 2.1 of !RFC9000}})) and states ({{Section 3 of !RFC9000}}) of QUIC
streams as closely as possible to aid in ease of implementation.

WebTransport streams are identified by a numeric value, or stream ID. Stream IDs
are only meaningful within the WebTransport session in which they were created.
They share the same semantics as QUIC stream IDs, with client initiated streams
having even-numbered stream IDs and server-initiated streams having
odd-numbered stream IDs. Similarly, they can be bidirectional or
unidirectional, indicated by the second least significant bit of the
stream ID.

Because WebTransport does not provide an acknowledgement mechanism for
WebTransport capsules, it relies on the underlying transport's in order delivery
to inform stream state transitions. Wherever QUIC relies on receiving an ack
for a packet to transition between stream states, WebTransport performs that
transition immediately.

## Use of Keying Material Exporters

WebTransport over HTTP/2 supports the use of TLS keying material exporters
{{Section 7.5 of !TLS=RFC8446}}. Since the underlying HTTP/2 connection could be
shared by multiple WebTransport sessions, WebTransport defines a mechanism for
deriving a TLS exporter that separates keying material for different sessions.
If the application requests an exporter for a given WebTransport session with a
specified label and context, the resulting exporter SHALL be a TLS exporter as
defined in {{Section 7.5 of TLS}} with the label set to
"EXPORTER-WebTransport" and the context set to the serialization of the
"WebTransport Exporter Context" struct as defined below.

~~~
WebTransport Exporter Context {
  WebTransport Session ID (64),
  WebTransport Application-Supplied Exporter Label Length (8),
  WebTransport Application-Supplied Exporter Label (8..),
  WebTransport Application-Supplied Exporter Context Length (8),
  WebTransport Application-Supplied Exporter Context (..)
}
~~~
{: #fig-wt-exporter-context title="WebTransport Exporter Context struct"}

A TLS exporter API might permit the context field to be omitted.  In this case,
as with TLS 1.3, the WebTransport Application-Supplied Exporter Context
becomes zero-length if omitted.

# WebTransport Capsules

WebTransport capsules mirror their QUIC frame counterparts as closely as
possible to enable maximal reuse of any applicable QUIC infrastructure by
implementors.

WebTransport capsules use the Capsule Protocol defined in {{Section 3.2 of
HTTP-DATAGRAM}}.

## PADDING Capsule {#PADDING}

*[PADDING]: #

A PADDING capsule is an HTTP capsule {{HTTP-DATAGRAM}} of type=0x190B4D38 and
has no semantic value. PADDING capsules can be used to introduce additional
data between other HTTP datagrams and can also be used to provide protection
against traffic analysis or for other reasons.

Note that, when used with WebTransport over HTTP/2, the PADDING capsule exists
alongside the ability to pad HTTP/2 frames ({{Section 10.7 of !RFC9113}}).
HTTP/2 padding is hop-by-hop and can be modified by intermediaries, while the
PADDING capsule traverses intermediaries.  The PADDING capsule cannot be smaller
than its own header, which means that the minimum size of the capsule is two
bytes: one byte for the Type and one byte to encode "0" for the Length.

~~~
PADDING Capsule {
  Type (i) = 0x190B4D38,
  Length (i),
  Padding (..),
}
~~~
{: #fig-padding title="PADDING Capsule Format"}

The Padding field MUST be set to an all-zero sequence of bytes of any length as
specified by the Length field.  A receiver is not obligated to verify padding
but MAY treat non-zero padding as a [stream error](#errors).

## WT_RESET_STREAM Capsule {#WT_RESET_STREAM}

*[WT_RESET_STREAM]: #

A WT_RESET_STREAM capsule is an HTTP capsule {{HTTP-DATAGRAM}} of
type=0x190B4D39 and allows either endpoint to abruptly terminate the sending
part of a WebTransport stream.

After sending a WT_RESET_STREAM capsule, an endpoint ceases transmission of
WT_STREAM capsules on the identified stream. A receiver of a WT_RESET_STREAM
capsule can discard any data in excess of the Reliable Size indicated, even if
that data was already received.

The WT_RESET_STREAM capsule follows the design of the QUIC RESET_STREAM_AT frame
{{!PARTIAL-RESET=I-D.ietf-quic-reliable-stream-reset}}.  Consequently, it
includes a Reliable Size field.  A WT_RESET_STREAM capsule MUST be sent after
WT_STREAM capsules that include an amount of data equal to or in excess of the
value in the Reliable Size field.  A receiver MUST treat the receipt of a
WT_RESET_STREAM with a Reliable Size smaller than the number of bytes it has
received on the stream as a session error.

~~~
WT_RESET_STREAM Capsule {
  Type (i) = 0x190B4D39,
  Length (i),
  Stream ID (i),
  Application Protocol Error Code (i),
  Reliable Size (i),
}
~~~
{: #fig-wt_reset_stream title="WT_RESET_STREAM Capsule Format"}

The WT_RESET_STREAM capsule defines the following fields:

   Stream ID:
   : A variable-length integer encoding of the WebTransport stream ID of the
     stream being terminated.

   Application Protocol Error Code:
   : A variable-length integer containing the application protocol error code
     that indicates why the stream is being closed.

   Reliable Size:
   : A variable-length integer indicating the amount of data that needs to be
     delivered to the application even though the stream is reset.

Unlike the equivalent QUIC frame, this capsule does not include a Final Size
field. In-order delivery of WT_STREAM capsules ensures that the amount of
session-level flow control consumed by a stream is always known by both
endpoints.

A WT_RESET_STREAM capsule MUST NOT be sent after a stream is closed or reset.
While QUIC permits redundant RESET_STREAM frames, the ordering guarantee in
HTTP/2 makes this unnecessary.  A [stream error](#errors) of type
WEBTRANSPORT_STREAM_STATE_ERROR MUST be sent if a WT_RESET_STREAM capsule is
received for a stream that is not in a valid state.

## WT_STOP_SENDING Capsule {#WT_STOP_SENDING}

*[WT_STOP_SENDING]: #

An HTTP capsule {{HTTP-DATAGRAM}} called WT_STOP_SENDING (type=0x190B4D3A) is
introduced to communicate that incoming data is being discarded on receipt per
application request. WT_STOP_SENDING requests that a peer cease transmission on
a WebTransport stream.

~~~
WT_STOP_SENDING Capsule {
  Type (i) = 0x190B4D3A,
  Length (i),
  Stream ID (i),
  Application Protocol Error Code (i),
}
~~~
{: #fig-wt_stop_sending title="WT_STOP_SENDING Capsule Format"}

The WT_STOP_SENDING capsule defines the following fields:

   Stream ID:
   : A variable-length integer carrying the WebTransport stream ID of the stream
     being ignored.

   Application Protocol Error Code:
   : A variable-length integer containing the application-specified reason the
     sender is ignoring the stream.

As defined in {{Section 3.5 of !RFC9000}}, the recipient of a WT_STOP_SENDING
capsule sends a WT_RESET_STREAM capsule in response, including the same error
code, if the stream is the "Ready" or "Send" state.

A WT_STOP_SENDING capsule MUST NOT be sent multiple times for the same stream.
While QUIC permits redundant STOP_SENDING frames, the ordering guarantee in
HTTP/2 makes this unnecessary.  A [stream error](#errors) of type
WEBTRANSPORT_STREAM_STATE_ERROR MUST be sent if a second WT_STOP_SENDING capsule
is received.

## WT_STREAM Capsule {#WT_STREAM}

*[WT_STREAM]: #

WT_STREAM capsules implicitly create a WebTransport stream and carry stream
data.

The Type field in the WT_STREAM capsule is either 0x190B4D3B or 0x190B4D3C.  The
least significant bit in the capsule type is the FIN bit (0x01), indicating
when set that the capsule marks the end of the stream in one direction.  Stream
data consists of any number of 0x190B4D3B capsules followed by a terminal
0x190B4D3C capsule.

~~~
WT_STREAM Capsule {
  Type (i) = 0x190B4D3B..0x190B4D3C,
  Length (i),
  Stream ID (i),
  Stream Data (..),
}
~~~
{: #fig-wt_stream title="WT_STREAM Capsule Format"}

WT_STREAM capsules contain the following fields:

Stream ID:
: The stream ID for the stream.  The second least significant bit of the Stream ID indicates whether
  the stream is bidirectional or unidirectional, as described in {{webtransport-streams}}.

Stream Data:
: Zero or more bytes of data for the stream.  Empty WT_STREAM capsules MUST NOT
  be used unless they open or close a stream; an endpoint MAY treat an empty
  WT_STREAM capsule that neither starts nor ends a stream as a session error.

A WT_STREAM capsule MUST NOT be sent after a stream is closed or reset.  While
QUIC permits redundant STREAM frames, the ordering guarantee in HTTP/2 makes
this unnecessary.  A [stream error](#errors) of type
WEBTRANSPORT_STREAM_STATE_ERROR MUST be sent if a WT_STREAM capsule is received
for a stream that is not in a valid state.

## WT_MAX_DATA Capsule {#WT_MAX_DATA}

*[WT_MAX_DATA]: #

An HTTP capsule {{HTTP-DATAGRAM}} called WT_MAX_DATA (type=0x190B4D3D)
({{Section 5.4.3 of WEBTRANSPORT-H3}}) is used to inform the peer of the maximum
amount of data that can be sent on the WebTransport session as a whole.

~~~
WT_MAX_DATA Capsule {
  Type (i) = 0x190B4D3D,
  Length (i),
  Maximum Data (i),
}
~~~
{: #fig-wt_max_data title="WT_MAX_DATA Capsule Format"}

WT_MAX_DATA capsules contain the following field:

   Maximum Data:
   : A variable-length integer indicating the maximum amount of data that can be
     sent on the entire connection, in units of bytes.

All data sent in WT_STREAM capsules counts toward this limit. The sum of the
lengths of Stream Data fields in WT_STREAM capsules MUST NOT exceed the value
advertised by a receiver.

The WT_MAX_DATA capsule defines special intermediary handling, as described in
{{Section 3.2 of HTTP-DATAGRAM}}.  Intermediaries MUST consume WT_MAX_DATA
capsules for flow control purposes and MUST generate and send appropriate flow
control signals for their limits; see {{flow-control-intermediaries}}.

The initial value for this limit MAY be communicated by sending a non-zero value
for SETTINGS_WT_INITIAL_MAX_DATA.

## WT_MAX_STREAM_DATA Capsule {#WT_MAX_STREAM_DATA}

*[WT_MAX_STREAM_DATA]: #

An HTTP capsule {{HTTP-DATAGRAM}} called WT_MAX_STREAM_DATA (type=0x190B4D3E) is
introduced to inform a peer of the maximum amount of data that can be sent on a
WebTransport stream.

~~~
WT_MAX_STREAM_DATA Capsule {
  Type (i) = 0x190B4D3E,
  Length (i),
  Stream ID (i),
  Maximum Stream Data (i),
}
~~~
{: #fig-wt_max_stream_data title="WT_MAX_STREAM_DATA Capsule Format"}

WT_MAX_STREAM_DATA capsules contain the following fields:

   Stream ID:
   : The stream ID of the affected WebTransport stream, encoded as a
     variable-length integer.

   Maximum Stream Data:
   : A variable-length integer indicating the maximum amount of data that can be
     sent on the identified stream, in units of bytes.

All data sent in WT_STREAM capsules for the identified stream counts toward this
limit. The sum of the lengths of Stream Data fields in WT_STREAM capsules on
the identified stream MUST NOT exceed the value advertised by a receiver.

The WT_MAX_STREAM_DATA capsule defines special intermediary handling, as
described in {{Section 3.2 of HTTP-DATAGRAM}}.  Intermediaries MUST consume
WT_MAX_STREAM_DATA capsules for flow control purposes and MUST generate and
send appropriate flow control signals for their limits; see
{{flow-control-intermediaries}}.

Initial values for this limit for unidirectional and bidirectional streams MAY
be communicated by sending non-zero values for
SETTINGS_WT_INITIAL_MAX_STREAM_DATA_UNI and
SETTINGS_WT_INITIAL_MAX_STREAM_DATA_BIDI respectively.

A WT_MAX_STREAM_DATA capsule MUST NOT be sent after a sender requests that a
stream be closed with WT_STOP_SENDING.  While QUIC permits redundant
MAX_STREAM_DATA frames, the ordering guarantee in HTTP/2 makes this unnecessary.
A [stream error](#errors) of type WEBTRANSPORT_STREAM_STATE_ERROR MUST be sent
if a WT_MAX_STREAM_DATA capsule is received after a WT_STOP_SENDING capsule for
the same stream.

## WT_MAX_STREAMS Capsule {#WT_MAX_STREAMS}

*[WT_MAX_STREAMS]: #

An HTTP capsule {{HTTP-DATAGRAM}} called WT_MAX_STREAMS is defined by {{Section
5.6.1 of WEBTRANSPORT-H3}} to inform the peer of the cumulative number of
streams of a given type it is permitted to open.  A WT_MAX_STREAMS capsule with
a type of 0x190B4D3F applies to bidirectional streams, and a WT_MAX_STREAMS
capsule with a type of 0x190B4D40 applies to unidirectional streams.

Note that, because Maximum Streams is a cumulative value representing the total
allowed number of streams, including previously closed streams, endpoints
repeatedly send new WT_MAX_STREAMS capsules with increasing Maximum Streams
values as streams are opened.

~~~
WT_MAX_STREAMS Capsule {
  Type (i) = 0x190B4D3F..0x190B4D40,
  Length (i),
  Maximum Streams (i),
}
~~~
{: #fig-wt_max_streams title="WT_MAX_STREAMS Capsule Format"}

WT_MAX_STREAMS capsules contain the following field:

   Maximum Streams:
   : A count of the cumulative number of streams of the corresponding type that
     can be opened over the lifetime of the connection. This value cannot
     exceed 2<sup>60</sup>, as it is not possible to encode stream IDs larger
     than 2<sup>62</sup>-1.

An endpoint MUST NOT open more streams than permitted by the current stream
limit set by its peer.  For instance, a server that receives a unidirectional
stream limit of 3 is permitted to open streams 3, 7, and 11, but not stream
15.

Note that this limit includes streams that have been closed as well as those
that are open.

The WT_MAX_STREAMS capsule defines special intermediary handling, as
described in {{Section 3.2 of HTTP-DATAGRAM}}.  Intermediaries MUST consume
WT_MAX_STREAMS capsules for flow control purposes and MUST generate and
send appropriate flow control signals for their limits.

Initial values for these limits MAY be communicated by sending non-zero values
for SETTINGS_WT_INITIAL_MAX_STREAMS_UNI and
SETTINGS_WT_INITIAL_MAX_STREAMS_BIDI.

## WT_DATA_BLOCKED Capsule {#WT_DATA_BLOCKED}

*[WT_DATA_BLOCKED]: #

A sender SHOULD send a WT_DATA_BLOCKED capsule (type=0x190B4D41) ({{Section
5.6.4 of WEBTRANSPORT-H3}}) when it wishes to send data but is unable to do so
due to WebTransport session-level flow control.  WT_DATA_BLOCKED capsules can be
used as input to tuning of flow control algorithms.

~~~
WT_DATA_BLOCKED Capsule {
  Type (i) = 0x190B4D41,
  Length (i),
  Maximum Data (i),
}
~~~
{: #fig-wt_data_blocked title="WT_DATA_BLOCKED Capsule Format"}

WT_DATA_BLOCKED capsules contain the following field:

   Maximum Data:
   : A variable-length integer indicating the session-level limit at which
     blocking occurred.

The WT_DATA_BLOCKED capsule defines special intermediary handling, as
described in {{Section 3.2 of HTTP-DATAGRAM}}.  Intermediaries MUST consume
WT_DATA_BLOCKED capsules for flow control purposes and MUST generate and
send appropriate flow control signals for their limits; see
{{flow-control-intermediaries}}.

## WT_STREAM_DATA_BLOCKED Capsule {#WT_STREAM_DATA_BLOCKED}

*[WT_STREAM_DATA_BLOCKED]: #

A sender SHOULD send a WT_STREAM_DATA_BLOCKED capsule (type=0x190B4D42) when it
wishes to send data but is unable to do so due to stream-level flow control.
This capsule is analogous to WT_DATA_BLOCKED.

~~~
WT_STREAM_DATA_BLOCKED Capsule {
  Type (i) = 0x190B4D42,
  Length (i),
  Stream ID (i),
  Maximum Stream Data (i),
}
~~~
{: #fig-wt_stream_data_blocked title="WT_STREAM_DATA_BLOCKED Capsule Format"}

WT_STREAM_DATA_BLOCKED capsules contain the following fields:

   Stream ID:
   : A variable-length integer indicating the WebTransport stream that is
     blocked due to flow control.

   Maximum Stream Data:
   : A variable-length integer indicating the offset of the stream at which the
     blocking occurred.

The WT_STREAM_DATA_BLOCKED capsule defines special intermediary handling, as
described in {{Section 3.2 of HTTP-DATAGRAM}}.  Intermediaries MUST consume
WT_STREAM_DATA_BLOCKED capsules for flow control purposes and MUST generate and
send appropriate flow control signals for their limits; see
{{flow-control-intermediaries}}.

A WT_STREAM_DATA_BLOCKED capsule MUST NOT be sent after a stream is closed or
reset.  While QUIC permits redundant STREAM_DATA_BLOCKED frames, the ordering
guarantee in HTTP/2 makes this unnecessary.  A [stream error](#errors) of type
WEBTRANSPORT_STREAM_STATE_ERROR MUST be sent if a WT_STREAM_DATA_BLOCKED capsule
is received for a stream that is not in a valid state.

## WT_STREAMS_BLOCKED Capsule {#WT_STREAMS_BLOCKED}

*[WT_STREAMS_BLOCKED]: #

A sender SHOULD send a WT_STREAMS_BLOCKED capsule (type=0x190B4D43 or
0x190B4D44) ({{Section 5.4.2 of WEBTRANSPORT-H3}}) when it wishes to open a
stream but is unable to do so due to the maximum stream limit set by its peer.
A WT_STREAMS_BLOCKED capsule of type 0x190B4D43 is used to indicate reaching the
bidirectional stream limit, and a STREAMS_BLOCKED capsule of type 0x190B4D44 is
used to indicate reaching the unidirectional stream limit.

A WT_STREAMS_BLOCKED capsule does not open the stream, but informs the peer that
a new stream was needed and the stream limit prevented the creation of the
stream.

~~~
WT_STREAMS_BLOCKED Capsule {
  Type (i) = 0x190B4D43..0x190B4D44,
  Length (i),
  Maximum Streams (i),
}
~~~
{: #fig-wt_streams_blocked title="WT_STREAMS_BLOCKED Capsule Format"}

WT_STREAMS_BLOCKED capsules contain the following field:

   Maximum Streams:
   : A variable-length integer indicating the maximum number of streams allowed
     at the time the capsule was sent. This value cannot exceed 2<sup>60</sup>,
     as it is not possible to encode stream IDs larger than 2<sup>62</sup>-1.

The WT_STREAMS_BLOCKED capsule defines special intermediary handling, as
described in {{Section 3.2 of HTTP-DATAGRAM}}.  Intermediaries MUST consume
WT_STREAMS_BLOCKED capsules for flow control purposes and MUST generate and
send appropriate flow control signals for their limits.

## DATAGRAM Capsule {#DATAGRAM_CAPSULE}

*[DATAGRAM_CAPSULE]: #

WebTransport over HTTP/2 uses the DATAGRAM capsule defined in {{Section 3.5 of
HTTP-DATAGRAM}} to carry datagram traffic.

~~~
DATAGRAM Capsule {
  Type (i) = 0x00,
  Length (i),
  HTTP Datagram Payload (..),
}
~~~
{: #fig-datagram title="DATAGRAM Capsule Format"}

When used in WebTransport over HTTP/2, DATAGRAM capsules contain the following
fields:

   HTTP Datagram Payload:
   : The content of the datagram to be delivered.

The data in DATAGRAM capsules is not subject to flow control. The receiver MAY
discard this data if it does not have sufficient space to buffer it.

An intermediary could forward the data in a DATAGRAM capsule over another
protocol, such as WebTransport over HTTP/3.  In QUIC, a datagram frame can span
at most one packet. Because of that, the applications have to know the maximum
size of the datagram they can send. However, when proxying the datagrams, the
hop-by-hop MTUs can vary.

{{Section 3.5 of HTTP-DATAGRAM}} indicates that intermediaries that forward
DATAGRAM capsules where QUIC datagrams {{DATAGRAM}} are available forward the
contents of the capsule as native QUIC datagrams, rather than as HTTP datagrams
in a DATAGRAM capsule. Similarly, when forwarding DATAGRAM capsules used as
part of a WebTransport over HTTP/2 session on a WebTransport session that
natively supports QUIC datagrams, such as WebTransport over HTTP/3
{{WEBTRANSPORT-H3}}, intermediaries follow the requirements in
{{WEBTRANSPORT-H3}} to use native QUIC datagrams.

## WT_CLOSE_SESSION Capsule {#WT_CLOSE_SESSION_CAPSULE}

*[WT_CLOSE_SESSION_CAPSULE]: #

WebTransport over HTTP/2 uses the WT_CLOSE_SESSION capsule defined in
{{Section 5 of WEBTRANSPORT-H3}} to terminate a WebTransport session with an
application error code and message.

WebTransport sessions can be terminated by optionally sending a WT_CLOSE_SESSION
capsule and then by closing the HTTP/2 stream associated with the session (see
{{errors}}).

~~~
WT_CLOSE_SESSION Capsule {
  Type (i) = WT_CLOSE_SESSION,
  Length (i),
  Application Error Code (32),
  Application Error Message (..8192),
}
~~~
{: #fig-wt_close_session title="WT_CLOSE_SESSION Capsule Format"}

When used in WebTransport over HTTP/2, WT_CLOSE_SESSION capsules contain the
following fields:

  Application Error Code:

  : A 32-bit error code provided by the application closing the connection.

  Application Error Message:

  : A UTF-8 encoded error message string provided by the application closing the
    connection.  The message takes up the remainder of the capsule, and its
    length MUST NOT exceed 1024 bytes.

An endpoint that sends a WT_CLOSE_SESSION capsule MUST then half-close the
stream by sending an HTTP/2 frame with the END_STREAM flag set ({{Section 5.1 of
HTTP2}}).  The recipient MUST close the stream upon receipt of the capsule by
replying with an HTTP/2 frame with the END_STREAM flag set; note that it does
not need to send a WT_CLOSE_SESSION capsule in response.

Cleanly terminating a WebTransport session without a WT_CLOSE_SESSION capsule
is semantically equivalent to terminating it with a WT_CLOSE_SESSION capsule
that has an error code of 0 and an empty error string.

## WT_DRAIN_SESSION Capsule {#WT_DRAIN_SESSION_CAPSULE}

*[WT_DRAIN_SESSION_CAPSULE]: #

HTTP/2 uses GOAWAY frames ({{Section 6.8 of HTTP2}}) to allow an endpoint to
gracefully stop accepting new streams while still finishing processing of
previously established streams.

WebTransport over HTTP/2 uses the WT_DRAIN_SESSION capsule defined in
{{Section 4.6 of WEBTRANSPORT-H3}} to gracefully shut down a WebTransport
session.

~~~
WT_DRAIN_SESSION Capsule {
  Type (i) = WT_DRAIN_SESSION,
  Length (i) = 0
}
~~~
{: #fig-wt_drain_session title="WT_DRAIN_SESSION Capsule Format"}

After sending or receiving either a WT_DRAIN_SESSION capsule or HTTP/2 GOAWAY
frame, an endpoint MAY continue using the session and MAY open new
WebTransport streams. The signal is intended for the application using
WebTransport, which is expected to attempt to gracefully terminate the
session as soon as possible.

## Capsule Ordering and Reliability

The use of an ordered and reliable transport means that a receiver does not need
to tolerate capsules that arrive out of order. This differs from QUIC in that a
receiver is required to treat the arrival of out of order frames rather than
being tolerant.

For an intermediary that forwards from an strongly-ordered transport (like
{{WEBTRANSPORT-H3}}) to a reliable transport (like this protocol), it is
necessary to maintain state for streams. A simple forwarding intermediary that
directly translates one type of protocol unit into another without understanding
the underlying state might cause a receiver to abort the session.

For instance, after a RESET_STREAM frame is forwarded, an intermediary cannot
forward a RESET_STREAM frame as a WT_RESET_STREAM capsule or a STREAM frame as a
WT_STREAM capsule without error.

# Requirements on TLS Usage

Because TLS keying material exporters are only secure for authentication when
they are uniquely bound to the TLS session {{!RFC7627}}, WebTransport requires
either one of the following conditions:

* The TLS version in use is greater than or equal to 1.3 {{TLS}}.

* The TLS version in use is 1.2, and the extended master secret extension
  {{RFC7627}} has been negotiated.

Clients MUST NOT send WebTransport over HTTP/2 requests on connections that do
not meet one of the two conditions above. If a server receives a WebTransport
over HTTP/2 request on a connection that meets neither, the server MUST treat
the request as malformed, as specified in {{Section 8.1.1 of HTTP2}}.

# Examples

An example of negotiating a WebTransport Stream on an HTTP/2 connection follows.
This example is intended to closely follow the example in {{Section 5.1 of
!RFC8441}} to help illustrate the differences defined in this document.

~~~
[[ From Client ]]                   [[ From Server ]]

SETTINGS

                                    SETTINGS
                                    SETTINGS_ENABLE_CONNECT_PROTOCOL = 1

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
Stream ID = 0
WebTransport Data

                                    WT_STREAM + FIN
                                    Stream ID = 0
                                    WebTransport Data

WT_STREAM + FIN
Stream ID = 0
WebTransport Data
~~~

An example of the server initiating a WebTransport Stream follows. The only
difference here is the endpoint that sends the first WT_STREAM capsule.

~~~
[[ From Client ]]                   [[ From Server ]]

SETTINGS

                                    SETTINGS
                                    SETTINGS_ENABLE_CONNECT_PROTOCOL = 1

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
                                    Stream ID = 1
                                    WebTransport Data

WT_STREAM + FIN
Stream ID = 1
WebTransport Data

                                    WT_STREAM + FIN
                                    Stream ID = 1
                                    WebTransport Data
~~~

# Considerations for Future Versions

Future versions of WebTransport that change the syntax of the CONNECT requests
used to establish WebTransport sessions will need to modify the upgrade token
used to identify WebTransport, allowing servers to offer multiple versions
simultaneously ({{Section 9.1 of WEBTRANSPORT-H3}}).

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

This document registers new HTTP/2 settings ({{h2-settings}}), HTTP/2 error
codes ({{iana-h2-error}}), new capsules ({{iana-capsules}}), and the
`WebTransport-Init` header field ({{iana-header}}).

## HTTP/2 SETTINGS Parameter Registration {#h2-settings}

The following entries are added to the "HTTP/2 Settings" registry established by
{{HTTP2}}:

{: anchor="SETTINGS_WT_INITIAL_MAX_DATA"}

The SETTINGS_WT_INITIAL_MAX_DATA parameter indicates the initial value for the
session data limit, otherwise communicated by the WT_MAX_DATA capsule
({{WT_MAX_DATA}}).  The default value for the SETTINGS_WT_INITIAL_MAX_DATA
parameter is "0", indicating that the endpoint needs to send a WT_MAX_DATA
capsule within each session before its peer is allowed to send any stream data
within that session.

Note that this limit applies to all WebTransport sessions that use the HTTP/2
connection on which this SETTING is sent.

Setting Name:

: SETTINGS_WT_INITIAL_MAX_DATA

Value:

: 0x2b61

Default:

: 0

Specification:

: This document

{: anchor="SETTINGS_WT_INITIAL_MAX_STREAM_DATA_UNI"}

The SETTINGS_WT_INITIAL_MAX_STREAM_DATA_UNI parameter indicates the initial
value for the stream data limit for incoming unidirectional streams, otherwise
communicated by the WT_MAX_STREAM_DATA capsule({{WT_MAX_STREAM_DATA}}).  The
default value for the SETTINGS_WT_INITIAL_MAX_STREAM_DATA_UNI parameter is "0",
indicating that the endpoint needs to send WT_MAX_STREAM_DATA capsules for each
stream within each individual WebTransport session before its peer is allowed
to send any stream data on those streams.

Note that this limit applies to all WebTransport streams on all sessions that
use the HTTP/2 connection on which this SETTING is sent.

Setting Name:

: SETTINGS_WT_INITIAL_MAX_STREAM_DATA_UNI

Value:

: 0x2b62

Default:

: 0

Specification:

: This document

{: anchor="SETTINGS_WT_INITIAL_MAX_STREAM_DATA_BIDI"}

The SETTINGS_WT_INITIAL_MAX_STREAM_DATA_BIDI parameter indicates the initial
value for the stream data limit for incoming data on bidirectional streams,
otherwise communicated by the WT_MAX_STREAM_DATA capsule
({{WT_MAX_STREAM_DATA}}).  The default value for the
SETTINGS_WT_INITIAL_MAX_STREAM_DATA_BIDI parameter is "0", indicating that the
endpoint needs to send WT_MAX_STREAM_DATA capsules for each stream within each
individual WebTransport session before its peer is allowed to send any stream
data on those streams.

Note that this limit applies to all WebTransport streams on all sessions that
use the HTTP/2 connection on which this SETTING is sent.

Setting Name:

: SETTINGS_WT_INITIAL_MAX_STREAM_DATA_BIDI

Value:

: 0x2b63

Default:

: 0

Specification:

: This document

{: anchor="SETTINGS_WT_INITIAL_MAX_STREAMS_UNI"}

The SETTINGS_WT_INITIAL_MAX_STREAMS_UNI parameter indicates the initial value
for the unidirectional max stream limit, otherwise communicated by the
WT_MAX_STREAMS capsule ({{WT_MAX_STREAMS}}).  The default value for the
SETTINGS_WT_INITIAL_MAX_STREAMS_UNI parameter is "0", indicating that the
endpoint needs to send WT_MAX_STREAMS capsules on each individual WebTransport
session before its peer is allowed to create any unidirectional streams within
that session.

Note that this limit applies to all WebTransport sessions that use the HTTP/2
connection on which this SETTING is sent.

Setting Name:

: SETTINGS_WT_INITIAL_MAX_STREAMS_UNI

Value:

: 0x2b64

Default:

: 0

Specification:

: This document

{: anchor="SETTINGS_WT_INITIAL_MAX_STREAMS_BIDI"}

The SETTINGS_WT_INITIAL_MAX_STREAMS_BIDI parameter indicates the initial value
for the bidirectional max stream limit, otherwise communicated by the
WT_MAX_STREAMS capsule ({{WT_MAX_STREAMS}}).  The default value for the
SETTINGS_WT_INITIAL_MAX_STREAMS_BIDI parameter is "0", indicating that the
endpoint needs to send WT_MAX_STREAMS capsules on each individual WebTransport
session before its peer is allowed to create any bidirectional streams within
that session.

Note that this limit applies to all WebTransport sessions that use the HTTP/2
connection on which this SETTING is sent.

Setting Name:

: SETTINGS_WT_INITIAL_MAX_STREAMS_BIDI

Value:

: 0x2b65

Default:

: 0

Specification:

: This document

## HTTP/2 Error Code Registration {#iana-h2-error}

The following entries are added to the "HTTP/2 Error Code" registry established
by {{HTTP2}}:

For WEBTRANSPORT_ERROR:

Code:
: 0xTBD

Name:
: WEBTRANSPORT_ERROR

Description:
: General WebTransport error detected

Reference:
: {{errors}}


For WEBTRANSPORT_STREAM_STATE_ERROR:

Code:
: 0xTBD

Name:
: WEBTRANSPORT_STREAM_STATE_ERROR

Description:
: Unexpected WebTransport stream-related capsule received

Reference:
: {{errors}}


## Capsule Types {#iana-capsules}

The following entries are added to the "HTTP Capsule Types" registry established
by {{HTTP-DATAGRAM}}:

The `PADDING` capsule.

Value:
: 0x190B4D38

Capsule Type:
: PADDING

Status:
: permanent

Specification:
: This document

Change Controller:
: IETF

Contact:
: WebTransport Working Group <webtransport@ietf.org>

Notes:
: None
{: spacing="compact"}

The `WT_RESET_STREAM` capsule.

Value:
: 0x190B4D39

Capsule Type:
: WT_RESET_STREAM

Status:
: permanent

Specification:
: This document

Change Controller:
: IETF

Contact:
: WebTransport Working Group <webtransport@ietf.org>

Notes:
: None
{: spacing="compact"}

The `WT_STOP_SENDING` capsule.

Value:
: 0x190B4D3A

Capsule Type:
: WT_STOP_SENDING

Status:
: permanent

Specification:
: This document

Change Controller:
: IETF

Contact:
: WebTransport Working Group <webtransport@ietf.org>

Notes:
: None
{: spacing="compact"}

The `WT_STREAM` capsule.

Value:
: 0x190B4D3B..0x190B4D3C

Capsule Type:
: WT_STREAM

Status:
: permanent

Specification:
: This document

Change Controller:
: IETF

Contact:
: WebTransport Working Group <webtransport@ietf.org>

Notes:
: None
{: spacing="compact"}

The `WT_MAX_STREAM_DATA` capsule.

Value:
: 0x190B4D3E

Capsule Type:
: WT_MAX_STREAM_DATA

Status:
: permanent

Specification:
: This document

Change Controller:
: IETF

Contact:
: WebTransport Working Group <webtransport@ietf.org>

Notes:
: None
{: spacing="compact"}

## HTTP Header Field Name {#iana-header}

IANA will register the following entry in the "Hypertext Transfer Protocol
(HTTP) Field Name Registry" maintained at
<https://www.iana.org/assignments/http-fields>:

Field Name:
: WebTransport-Init

Template:
: None

Status:
: permanent

Reference:
: This document

Comments:
: None

--- back

# Acknowledgments
{:numbered="false"}

Thanks to Anthony Chivetta, Eric Gorbaty, Ankshit Jain, Joshua Otto, and
Valentin Pistol for their contributions in the design and implementation of
this work. The requirements on TLS usage were inspired by {{?RFC9729}}.
