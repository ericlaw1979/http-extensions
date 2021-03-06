---
title: HTTP Client Hints
abbrev:
docname: draft-ietf-httpbis-client-hints-latest
date: 2016
category: std

ipr: trust200902
area: Applications and Real-Time
workgroup: HTTP
keyword: Internet-Draft
keyword: client hints
keyword: conneg
keyword: Content Negotiation

stand_alone: yes
pi: [toc, tocindent, sortrefs, symrefs, strict, compact, subcompact, comments, inline]

author:
 -
    ins: I. Grigorik
    name: Ilya Grigorik
    organization: Google
    email: ilya@igvita.com
    uri: https://www.igvita.com/

normative:
  RFC2119:
  RFC5234:
  RFC7230:
  RFC7231:
  RFC7234:
  I-D.ietf-httpbis-key:
  NETINFO:
    target: https://w3c.github.io/netinfo/
    title: "Network Information API"
    date: 2015-12
    author:
    -
      ins: M. Cáceres
      name: Marcos Cáceres
      organization:  Mozilla Corporation
    -
      ins: F.J. Moreno
      name: Fernando Jiménez Moreno
      organization: Telefonica
    -
      ins: I. Grigorik
      name: Ilya Grigorik
      organization: Google
  W3C.REC-html5-20141028:
  W3C.CR-css-values-3-20150611:
  CSS2:
    target: http://www.w3.org/TR/2011/REC-CSS2-20110607
    title: "Cascading Style Sheets Level 2 Revision 1 (CSS 2.1) Specification"
    date: 2011-06
    author:
    -
      ins: B. Bos
    -
      ins: T. Celic
    -
      ins: I. Hickson
    -
      ins: H. W. Lie
    seriesinfo:
      "W3C Recommendation": REC-CSS2-20110607

informative:
  RFC6265:

--- abstract

An increasing diversity of Web-connected devices and software capabilities has created a need to deliver optimized content for each device.

This specification defines a set of HTTP request header fields, colloquially known as Client Hints, to address this. They are intended to be used as input to proactive content negotiation; just as the Accept header field allows clients to indicate what formats they prefer, Client Hints allow clients to indicate a list of device and agent specific preferences.


--- note_Note_to_Readers

Discussion of this draft takes place on the HTTP working group mailing list
(ietf-http-wg@w3.org), which is archived at <https://lists.w3.org/Archives/Public/ietf-http-wg/>.

Working Group information can be found at <http://httpwg.github.io/>; source
code and issues list for this draft can be found at <https://github.com/httpwg/http-extensions/labels/client-hints>.


--- middle

# Introduction

There are thousands of different devices accessing the web, each with different device capabilities and preference information. These device capabilities include hardware and software characteristics, as well as dynamic user and client preferences.

One way to infer some of these capabilities is through User-Agent (UA; Section 5.5.3 of {{RFC7231}}) detection against an established database of client signatures. However, this technique requires acquiring such a database, integrating it into the serving path, and keeping it up to date. However, even once this infrastructure is deployed, UA sniffing has numerous limitations:

  - UA detection cannot reliably identify all static variables
  - UA detection cannot infer any dynamic client preferences
  - UA detection requires an external device database
  - UA detection is not cache friendly

A popular alternative strategy is to use HTTP cookies ({{RFC6265}}) to communicate some information about the client. However, this approach is also not cache friendly, bound by same origin policy, and imposes additional client-side latency by requiring JavaScript execution to create and manage HTTP cookies.

This document defines a set of new request header fields that allow the client to perform proactive content negotiation (Section 3.4.1 of {{RFC7231}}) by indicating a list of device and agent specific preferences, through a mechanism similar to the Accept header field which is used to indicate preferred response formats.

Client Hints does not supersede or replace the User-Agent header field. Existing device detection mechanisms can continue to use both mechanisms if necessary. By advertising its capabilities within a request header field, Client Hints allows for cache friendly and proactive content negotiation.

## Notational Conventions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in {{RFC2119}}.

This document uses the Augmented Backus-Naur Form (ABNF) notation of {{RFC5234}} with the list rule extension defined in {{RFC7230}}, Appendix B. It includes by reference the DIGIT rule from {{RFC5234}} and the OWS and field-name rules from {{RFC7230}}.


# Client Hint Request Header Fields

A Client Hint request header field is a HTTP header field that is used by HTTP clients to indicate configuration data that can be used by the server to select an appropriate response. Each one conveys a list of client preferences that the server can use to adapt and optimize the response.

## Sending Client Hints

Clients control which Client Hint headers and their respective header fields are communicated, based on their default settings, user configuration and/or preferences. The user can be given the choice to enable, disable, or override specific hints.

The client and server, or an intermediate proxy, can use an opt-in mechanism to negotiate which fields should be reported to allow for efficient content adaption.


## Server Processing of Client Hints

Servers respond with an optimized response based on one or more received hints from the client. When doing so, and if the resource is cacheable, the server MUST also emit a Vary response header field (Section 7.1.4 of {{RFC7231}}), and optionally Key ({{I-D.ietf-httpbis-key}}), to indicate which hints can affect the selected response and whether the selected response is appropriate for a later request.

Further, depending on the used hint, the server can emit additional response header fields to confirm the property of the response, such that the client can adjust its processing. For example, this specification defines "Content-DPR" response header field that needs to be returned by the server when the "DPR" hint is used to select the response.


### Advertising Support for Client Hints {#accept-ch}

Servers can advertise support for Client Hints using the Accept-CH header field or an equivalent HTML meta element with http-equiv attribute ({{W3C.REC-html5-20141028}}).

~~~ abnf7230
  Accept-CH = #field-name
~~~

For example:

~~~ example
  Accept-CH: DPR, Width, Viewport-Width, Downlink
~~~

When a client receives Accept-CH, or if it is capable of processing the HTML response and finds an equivalent HTML meta element, it SHOULD append the Client-Hint header fields that match the advertised field-values to the header list of all subsequent requests. For example, based on Accept-CH example above, a user agent could append DPR, Width, Viewport-Width, and Downlink header fields to all subresource requests initiated by the page constructed from the response. Alternatively, a client can treat advertised support as a persistent origin preference and append same header fields on all future requests initiated to and by the resources associated with that origin.

### Interaction with Caches

When selecting an optimized response based on one or more Client Hints, and if the resource is cacheable, the server needs to emit a Vary response header field ({{RFC7234}}) to indicate which hints can affect the selected response and whether the selected response is appropriate for a later request.

~~~ example
  Vary: DPR
~~~

Above example indicates that the cache key needs to include the DPR header field.

~~~ example
  Vary: DPR, Width, Downlink
~~~

Above example indicates that the cache key needs to include the DPR, Width, and Downlink header fields.

Client Hints MAY be combined with Key ({{I-D.ietf-httpbis-key}}) to enable fine-grained control of the cache key for improved cache efficiency. For example, the server can return the following set of instructions:

~~~ example
  Key: DPR;partition=1.5:2.5:4.0
~~~

Above example indicates that the cache key needs to include the value of the DPR header field with three segments: less than 1.5, 1.5 to less than 2.5, and 4.0 or greater.

~~~ example
  Key: Width;div=320
~~~

Above example indicates that the cache key needs to include the value of the Width header field and be partitioned into groups of 320: 0-320, 320-640, and so on.

~~~ example
  Key: Downlink;partition=0.5:1.0:3.0:5.0:10
~~~

Above example indicates that the cache key needs to include the (Mbps) value of the Downlink header field with six segments: less than 0.5, 0.5 to less than 1.0, 1.0 to less than 3.0, 3.0 to less than 5.0, 5.0 to less than 10; 10 or higher.


# The DPR Client Hint {#dpr}

The "DPR" request header field is a number that indicates the client's current Device Pixel Ratio (DPR), which is the ratio of physical pixels over CSS px (Section 5.2 of {{W3C.CR-css-values-3-20150611}}) of the layout viewport (Section 9.1.1 of [CSS2]) on the device.

~~~ abnf7230
  DPR = 1*DIGIT [ "." 1*DIGIT ]
~~~

If DPR occurs in a message more than once, the last value overrides all previous occurrences.


## Confirming Selected DPR {#content-dpr}

The "Content-DPR" response header field is a number that indicates the ratio between physical pixels over CSS px of the selected image response.

~~~ abnf7230
  Content-DPR = 1*DIGIT [ "." 1*DIGIT ]
~~~

DPR ratio affects the calculation of intrinsic size of image resources on the client - i.e. typically, the client automatically scales the natural size of the image by the DPR ratio to derive its display dimensions. As a result, the server MUST explicitly indicate the DPR of the selected image response whenever the DPR hint is used, and the client MUST use the DPR value returned by the server to perform its calculations. In case the server returned Content-DPR value contradicts previous client-side DPR indication, the server returned value MUST take precedence.

Note that DPR confirmation is only required for image responses, and the server does not need to confirm the resource width as this value can be derived from the resource itself once it is decoded by the client.

If Content-DPR occurs in a message more than once, the last value overrides all previous occurrences.


# The Width Client Hint {#width}

The "Width" request header field is a number that indicates the desired resource width in physical px (i.e. intrinsic size of an image). The provided physical px value is a number rounded to the smallest following integer (i.e. ceiling value).

~~~ abnf7230
  Width = 1*DIGIT
~~~

If the desired resource width is not known at the time of the request or the resource does not have a display width, the Width header field can be omitted. If Width occurs in a message more than once, the last value overrides all previous occurrences.


# The Viewport-Width Client Hint {#viewport-width}

The "Viewport-Width" request header field is a number that indicates the layout viewport width in CSS px. The provided CSS px value is a number rounded to the smallest following integer (i.e. ceiling value).

~~~ abnf7230
  Viewport-Width = 1*DIGIT
~~~

If Viewport-Width occurs in a message more than once, the last value overrides all previous occurrences.


# The Downlink Client Hint {#downlink}

The "Downlink" request header field is a number that indicates the client's maximum downlink speed in megabits per second (Mbps), as defined by the "downlinkMax" attribute in the W3C Network Information API ({{NETINFO}}).

~~~ abnf7230
  Downlink = 1*DIGIT [ "." 1*DIGIT ]
~~~

If Downlink occurs in a message more than once, the minimum value should be used to override other occurrences.


# The Save-Data Client Hint {#save-data}

The "Save-Data" request header field consists of one or more tokens that indicate client's preference for reduced data usage, due to high transfer costs, slow connection speeds, or other reasons.

~~~ abnf7230
  Save-Data = sd-token *( OWS ";" OWS [sd-token] )
  sd-token = token
~~~

This document defines the "on" sd-token value, which is used as a signal indicating explicit user opt-in into a reduced data usage mode on the client, and when communicated to origins allows them to deliver alternate content honoring such preference - e.g. smaller image and video resources, alternate markup, and so on. New token and extension token values can only be defined by revisions of this specification.


# Examples

For example, given the following request header fields:

~~~ example
  DPR: 2.0
  Width: 320
  Viewport-Width: 320
~~~

The server knows that the device pixel ratio is 2.0, that the intended display width of the requested resource is 160 CSS px (320 physical pixels at 2x resolution), and that the viewport width is 320 CSS px.

If the server uses above hints to perform resource selection for an image asset, it must confirm its selection via the Content-DPR response header to allow the client to calculate the appropriate intrinsic size of the image response. The server does not need to confirm resource width, only the ratio between physical pixels and CSS px of the selected image resource:

~~~ example
  Content-DPR: 1.0
~~~

The Content-DPR response header field indicates to the client that the server has selected resource with DPR ratio of 1.0. The client can use this information to perform additional processing on the resource - for example, calculate the appropriate intrinsic size of the image resource such that it is displayed at the correct resolution.

Alternatively, the server could select an alternate resource based on the maximum downlink speed advertised in the request header fields:

~~~ example
  Downlink: 0.384
~~~

The server knows that the client's maximum downlink speed is 0.384Mbps (GPRS EDGE), and it can use this information to select an optimized resource - for example, an alternate image asset, stylesheet, HTML document, media stream, and so on.


# Security Considerations

Client Hints defined in this specification do not expose any new information about the user's environment beyond what is already available to, and can be communicated by, the application at runtime via JavaScript - e.g. viewport and image display width, device pixel ratio, and so on.

However, implementors should consider the privacy implications of various methods to enable delivery of Client Hints - see "Sending Client Hints" section. For example, sending Client Hints on all requests can make information about the user's environment available to origins that otherwise did not have access to this data (e.g. origins hosting non-script resources), which might or not be the desired outcome. The implementors can provide mechanisms to control such behavior via explicit opt-in, or other mechanisms. Similarly, the implementors should consider how and whether delivery of Client Hints is affected when the user is in "incognito" or similar privacy mode.


# IANA Considerations

This document defines the "Accept-CH", "DPR", "Width", and "Downlink" HTTP request fields, "Content-DPR" HTTP response field, and registers them in the Permanent Message Header Fields registry.

## Accept-CH {#iana-accept-ch}
- Header field name: Accept-CH
- Applicable protocol: HTTP
- Status: standard
- Author/Change controller: IETF
- Specification document(s): {{accept-ch}}
- Related information: for Client Hints

## Content-DPR {#iana-content-dpr}
- Header field name: Content-DPR
- Applicable protocol: HTTP
- Status: standard
- Author/Change controller: IETF
- Specification document(s): {{content-dpr}} of this document
- Related information: for Client Hints

## Downlink {#iana-downlink}
- Header field name: Downlink
- Applicable protocol: HTTP
- Status: standard
- Author/Change controller: IETF
- Specification document(s): {{downlink}} of this document
- Related information: for Client Hints

## DPR {#iana-dpr}
- Header field name: DPR
- Applicable protocol: HTTP
- Status: standard
- Author/Change controller: IETF
- Specification document(s): {{dpr}} of this document
- Related information: for Client Hints

## Save-Data {#iana-save-data}
- Header field name: Save-Data
- Applicable protocol: HTTP
- Status: standard
- Author/Change controller: IETF
- Specification document(s): {{save-data}} of this document
- Related information: for Client Hints

## Viewport-Width {#iana-viewport-width}
- Header field name: Viewport-Width
- Applicable protocol: HTTP
- Status: standard
- Author/Change controller: IETF
- Specification document(s): {{viewport-width}} of this document
- Related information: for Client Hints

## Width {#iana-width}
- Header field name: Width
- Applicable protocol: HTTP
- Status: standard
- Author/Change controller: IETF
- Specification document(s): {{width}} of this document
- Related information: for Client Hints

--- back

# Changes

## Since -00

* Issue 168 (make Save-Data extensible) updated ABNF.
* Issue 163 (CH review feedback) editorial feedback from httpwg list.
* Issue 153 (NetInfo API citation) added normative reference.


## Since -01

None yet.
