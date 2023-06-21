---
title: "Using Service Bindings with DANE"
abbrev: "SVCB-DANE"
category: std

docname: draft-ietf-dnsop-svcb-dane-latest
ipr: trust200902
area: ops
workgroup: dnsop
keyword: Internet-Draft
updates: rfc6698

stand_alone: yes
smart_quotes: no
pi: [toc, sortrefs, symrefs]

author:
 -
    name: Benjamin M. Schwartz
    organization: Meta Platforms, Inc.
    email: ietf@bemasc.net
 -
    name: Robert Evans
    organization: Google LLC
    email: evansr@google.com

normative:

informative:


--- abstract

Service Binding records introduce a new form of name indirection in DNS. This document specifies DNS-Based Authentication of Named Entities (DANE) interaction with Service Bindings to secure endpoints including use of ports and transports discovered via Service Parameters.

--- middle

# Introduction

The DNS-Based Authentication of Named Entities specification {{!RFC7671}} explains how clients locate the TLSA record for a service of interest, starting with knowledge of the service's hostname, transport, and port number.  These are concatenated, forming a name like _8080._tcp.example.com.  It also specifies how clients should locate the TLSA record when one or more CNAME records are present, aliasing either the hostname or the TLSA record's name, and the resulting server names used in TLS.

There are various DNS records other than CNAME that add indirection to the host resolution process, requiring similar specifications.  Thus, {{?RFC7672}} describes how DANE interacts with MX records, and {{?RFC7673}} describes its interaction with SRV records.

This draft describes the interaction of DANE with indirection via Service Bindings {{!SVCB=I-D.draft-ietf-dnsop-svcb-https}}, i.e. SVCB-compatible records such as SVCB and HTTPS.  It also explains how to use DANE with new TLS-based transports such as QUIC.

# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Using DANE with Service Bindings

{{Section 6 of RFC7671}} says:

> With protocols that support explicit transport redirection via DNS MX records, SRV records, or other similar records, the TLSA base domain is based on the redirected transport endpoint rather than the origin domain.

This draft applies the same logic to SVCB-compatible records.  Specifically, if SVCB resolution was entirely secure (including any AliasMode records and/or CNAMEs), then for each connection attempt derived from a SVCB-compatible record,

* The initial TLSA base domain MUST be the final SVCB TargetName used for this connection attempt.  (Names appearing earlier in a resolution chain are not used.)
* The transport prefix MUST be the transport of this connection attempt (possibly influenced by the "alpn" SvcParam).
* The port prefix MUST be the port number of this connection attempt (possibly influenced by the "port" SvcParam).

If the initial TLSA base domain is the start of a secure CNAME chain, clients MUST first try to use the end of the chain as the TLSA base domain, with fallback to the initial base domain, as described in {{Section 7 of RFC7671}}.

If any TLSA QNAME is aliased by a CNAME, clients MUST follow the TLSA CNAME to complete the resolution of the TLSA record.  (This does not alter the TLSA base domain.)

If a TLSA RRSet is securely resolved, the client MUST set the SNI to the TLSA base domain of the RRSet.  In usage modes other than DANE-EE(3), the client MUST validate that the certificate covers this base domain, and MUST NOT require it to cover any other domain.

If the client has SVCB-optional behavior (as defined in {{Section 3 of SVCB}}), it MUST use the standard DANE logic described in {{Section 4.1 of RFC6698}} when falling back to non-SVCB connection.


# Updating the TLSA protocol prefixes {#protocols}

{{Section 3 of !RFC6698}} defined the protocol prefix used for constructing TLSA QNAMEs, and said:

> The transport names defined for this protocol are "tcp", "udp", and "sctp".

At that time, there was exactly one TLS-based protocol defined for each of these transports.  However, with the introduction of QUIC {{!RFC9000}}, there are now multiple TLS-derived protocols that can operate over UDP, even on the same port.  To distinguish the availability and configuration of DTLS and QUIC, this draft Updates the above sentence as follows:

> The transport names defined for this protocol are "tcp" (TLS over TCP {{!RFC8446}}), "udp" (DTLS {{!I-D.draft-ietf-tls-dtls13}}), "sctp" (TLS over SCTP {{!RFC3436}}), and "quic" (QUIC {{!RFC9000}}).

# Operational considerations

## Recommended configurations

Service consumers are expected to use CNAME or SVCB AliasMode to point at provider-controlled records, e.g.:

~~~
alias.net.                  HTTPS 0 xyz.provider.com.
www.alias.net.              CNAME xyz.provider.com.
xyz.provider.com.           HTTPS 1 . alpn=h2 ...
xyz.provider.com.           A     192.0.2.1
_443._tcp.xyz.provider.com. TLSA  <provider keys>
~~~

For ease of management, providers may want to alias various TLSA QNAMEs to a single RRSet:

~~~
_443._tcp.xyz.provider.com. CNAME dane-central.provider.com.
dane-central.provider.com.  TLSA  <provider keys>
~~~

## Accidental pinning

When a service is used by third-party consumers, DANE allows the consumer to publish records that make claims about the certificates used by the service.  When the service subsequently rotates its TLS keys, DANE authentication will fail for these consumers, resulting in an outage.  Accordingly, zone owners MUST NOT publish TLSA records for public keys that are not under their control unless they have an explicit arrangement with the key holder.

To prevents the above misconfiguration and ensure that TLS keys can be rotated freely, service operators MAY reject TLS connections whose SNI does not correspond to an approved TLSA base domain.

Service Bindings also enable any third party consumer to publish fixed SvcParams for the service.  This can cause an outage or service degradation if the service makes a backward-incompatible configuration change.  Accordingly, zone owners SHOULD NOT publish SvcParams for a TargetName that they do not control, and service operators should take caution when making incompatible configuration changes.


# Security Considerations

This document specifies the use of TLSA as a property of each connection attempt.  In environments where DANE is optional, this means that the fallback procedure might use DANE for some conection attempts but not others.

This document only specifies the use of TLSA records when the SVCB records were resolved securely.  Use of TLSA records in conjunction with insecurely resolved SVCB records is not safe in general, although there may be some configurations where it is appropriate (e.g. when only opportunistic security is available).


# Examples

The following examples demonstrate Service Binding interaction with TLSA base domain selection.

All of the RRSets below are assumed fully-secure with all related DNSSEC record types omitted for brevity.

## HTTPS ServiceMode

Given service URI `https://api.example.com` and record:

~~~
api.example.com. HTTPS 1 .
~~~

The TLSA QNAME is `_443._tcp.api.example.com`.

## HTTPS AliasMode

Given service URI `https://api.example.com` and records:

~~~
api.example.com.     HTTPS 0 svc4.example.net.
svc4.example.net.    HTTPS 0 xyz.example-cdn.com.
xyz.example-cdn.com. A     192.0.2.1
~~~

The TLSA QNAME is `_443._tcp.xyz.example-cdn.com`.

## QUIC and CNAME

Given service URI `https://api.example.com` and records:

~~~
www.example.com.  CNAME api.example.com.
api.example.com.  HTTPS 1 svc4.example.net alpn=h2,h3 port=8443
svc4.example.net. CNAME xyz.example-cdn.com.
~~~

If the connection attempt is using HTTP/3, the transport label is set to `_quic`; otherwise `_tcp` is used.

The initial TLSA QNAME would be one of:

* `_8443._quic.xyz.example-cdn.com`
* `_8443._tcp.xyz.example-cdn.com`

If no TLSA record is found, the fallback TLSA QNAME would be one of:

* `_8443._quic.svc4.example.net`
* `_8443._tcp.svc4.example.net`

## New scheme ServiceMode

Given service URI `foo://api.example.com:8443` and record:

~~~
_8443._foo.api.example.com. SVCB 1 api.example.com.
~~~

The TLSA QNAME is `_8443._$PROTO.api.example.com`, where $PROTO is the appropriate value for the client-selected transport as discussed in {{protocols}} .

## New scheme AliasMode

Given service URI `foo://api.example.com:8443` and records:

~~~
_8443._foo.api.example.com. SVCB 0 svc4.example.net.
svc4.example.net.           SVCB 1 .
svc4.example.net.           A    192.0.2.1
~~~

The TLSA QNAME is `_8443._$PROTO.svc4.example.net` (with $PROTO as above).  This is the same if the ServiceMode record is absent.

## New protocols

Given service URI `foo://api.example.com:8443` and records:

~~~
_8443._foo.api.example.com. SVCB 0 svc4.example.net.
svc4.example.net. SVCB 3 . alpn=foo,bar port=8004
~~~

The TLSA QNAME is `_8004._$PROTO1.svc4.example.net` or `_8004._$PROTO2.svc4.example.net`, where $PROTO1 and $PROTO2 are the transport prefixes appropriate for "foo" and "bar" respectively.  (Note that SVCB requires each ALPN to unambiguously indicate a transport.)

## DNS ServiceMode

Given a DNS server `dns.example.com` and record:

~~~
_dns.dns.example.com. SVCB 1 dns.example.com. alpn=dot
~~~

The TLSA QNAME is `_853._tcp.dns.example.com`.  The TLSA base name is taken from the SVCB TargetName.  The port and protocol are taken from the "dot" ALPN value.

## DNS AliasMode

Given a DNS server `dns.example.com` and records:

~~~
_dns.dns.example.com. SVCB 0 dns.my-dns-host.net.
dns.my-dns-host.net.  SVCB 1 . alpn=dot
~~~

The TLSA QNAME is `_853._tcp.ns1.my-dns-host.net`.


# IANA Considerations

IANA is instructed to add the following entry to the "Underscored and Globally Scoped DNS Node Names" registry:

| RR Type | _NODE NAME | Reference       |
| ------- | ---------- | --------------- |
| TLSA    | _quic      | (This document) |

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
