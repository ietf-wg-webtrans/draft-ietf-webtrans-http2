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

Introduction


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


