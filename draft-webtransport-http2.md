---
title: "WebTransport using HTTP/2"
abbrev: "WebTransportHTTP/2"
docname: draft-webtransport-http2-latest
date: {DATE}
category: std

area: art
workgroup: webtrans
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: F. Last
    name: First Last
    organization: Company
    email: first.last@company.com

--- abstract

WebTransport [OVERVIEW] is a protocol framework that enables clients
   constrained by the Web security model to communicate with a remote
   server using a secure multiplexed transport.  This document describes
   Http3Transport, a WebTransport protocol that is based on HTTP/3
   [HTTP3] and provides support for unidirectional streams,
   bidirectional streams and datagrams, all multiplexed within the same
   HTTP/3 connection.

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

TODO Introduction


# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.


# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.



--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.


