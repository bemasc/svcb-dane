---
title: "Using DNSSEC Authentication of Named Entities (DANE) with DNS Service Bindings (SVCB) and QUIC"
abbrev: "SVCB-DANE"
category: std

docname: draft-ietf-dnsop-svcb-dane-latest
ipr: trust200902
area: ops
workgroup: dnsop
keyword: Internet-Draft
updates: 6698

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

Service Binding (SVCB) records introduce a new form of name indirection in DNS.  They also convey information about the endpoint's supported protocols, such as whether QUIC transport is available.  This document specifies how DNS-Based Authentication of Named Entities (DANE) interacts with Service Bindings to secure endpoints, including use of port numbers and QUIC support discovered via SVCB queries.

--- middle

# Introduction

The DNS-Based Authentication of Named Entities specification {{!RFC7671}} explains how clients locate the TLSA record for a service of interest, starting with knowledge of the service's hostname, transport, and port number.  These are concatenated, forming a name like `_8080._tcp.example.com`.  It also specifies how clients should locate the TLSA records when one or more CNAME records are present, aliasing either the hostname or the initial TLSA query name, and the resulting server names used in TLS.

There are various DNS records other than CNAME that add indirection to the host resolution process, requiring similar specifications.  Thus, {{?RFC7672}} describes how DANE interacts with MX records, and {{?RFC7673}} describes its interaction with SRV records.

This document describes the interaction of DANE with indirection via Service Bindings {{!SVCB=I-D.draft-ietf-dnsop-svcb-https}}, i.e. SVCB-compatible records such as SVCB and HTTPS.  It also explains how to use DANE with new TLS-based transports such as QUIC.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

The contents of this document apply equally to all SVCB-compatible record types, such as SVCB and HTTPS records.  For brevity, the abbrevation "SVCB" is used to refer to these record types generally.

# Using DANE with Service Bindings (SVCB)

{{Section 6 of RFC7671}} says:

> With protocols that support explicit transport redirection via DNS MX records, SRV records, or other similar records, the TLSA base domain is based on the redirected transport endpoint rather than the origin domain.

This document applies the same logic to SVCB-compatible records.  Specifically, if SVCB resolution was entirely secure (including any AliasMode records and/or CNAMEs), then for each connection attempt derived from a SVCB-compatible record,

* The initial TLSA base domain MUST be the final SVCB TargetName used for this connection attempt.  (Names appearing earlier in a resolution chain are not used.)
* The transport prefix MUST be the transport of this connection attempt (possibly influenced by the "alpn" SvcParam).
* The port prefix MUST be the port number of this connection attempt (possibly influenced by the "port" SvcParam).

Resolution security is assessed according to the criteria in {{Section 4.1 of !RFC6698}}.

If the initial TLSA base domain is the start of a secure CNAME chain, clients MUST first try to use the end of the chain as the TLSA base domain, with fallback to the initial base domain, as described in {{Section 7 of RFC7671}}.

If any TLSA QNAME is aliased by a CNAME, clients MUST follow the TLSA CNAME to complete the resolution of the TLSA record.  (This does not alter the TLSA base domain.)

If a TLSA RRSet is securely resolved, the client MUST set the SNI to the TLSA base domain of the RRSet.  In usage modes other than DANE-EE(3), the client MUST validate that the certificate covers this base domain, and MUST NOT require it to cover any other domain.

If the client has SVCB-optional behavior (as defined in {{Section 3 of SVCB}}), it MUST use the standard DANE logic described in {{Section 4.1 of RFC6698}} when falling back to non-SVCB connection.


# Adding a TLSA protocol prefix for QUIC {#protocols}

{{Section 3 of !RFC6698}} defined the protocol prefix used for constructing TLSA QNAMEs, and said:

> The transport names defined for this protocol are "tcp", "udp", and "sctp".

At that time, there was exactly one TLS-based protocol defined for each of these transports.  However, with the introduction of QUIC {{!RFC9000}}, there are now multiple TLS-derived protocols that can operate over UDP, even on the same port.  To distinguish the availability and configuration of DTLS and QUIC, this document updates the above sentence as follows:

> The transport names defined for this protocol are "tcp" (TLS over TCP {{!RFC8446}}), "udp" (DTLS {{!RFC9147}}), "sctp" (TLS over SCTP {{!RFC3436}}), and "quic" (QUIC {{!RFC9000}}).

# Operational considerations

## Recommended configurations

Service consumers are expected to use a CNAME or SVCB AliasMode record to point at provider-controlled records when possible, e.g.:

~~~ Zone
alias.example.                  HTTPS 0 xyz.provider.example.
www.alias.example.              CNAME xyz.provider.example.
xyz.provider.example.           HTTPS 1 . alpn=h2 ...
xyz.provider.example.           A     192.0.2.1
_443._tcp.xyz.provider.example. TLSA  ...
~~~

If the service needs its own SvcParamKeys, it cannot use CNAME or AliasMode, so it publishes its own SVCB ServiceMode record with SvcParams that are compatible with the provider, e.g.:

~~~ Zone
_dns.dns.example. HTTPS 1 xyz.provider.example. ( alpn=h2 ...
                                                  dohpath=/doh{?dns} )
~~~

For ease of management, providers may want to alias various TLSA QNAMEs to a single RRSet:

~~~ Zone
_443._tcp.xyz.provider.example. CNAME dane-central.provider.example.
dane-central.provider.example.  TLSA  ...
~~~

Any DANE certificate usage mode is compatible with SVCB, but the usage guidelines from {{Section 4 of !RFC7671}} continue to apply.

## Accidental pinning

When a service is used by third-party consumers, DANE allows the consumer to publish records that make claims about the certificates used by the service.  When the service subsequently rotates its TLS keys, DANE authentication will fail for these consumers, resulting in an outage.  Accordingly, zone owners MUST NOT publish TLSA records for public keys that are not under their control unless they have an explicit arrangement with the key holder.

To prevents the above misconfiguration and ensure that TLS keys can be rotated freely, service operators MAY reject TLS connections whose SNI does not correspond to an approved TLSA base domain.

Service Bindings also enable any third party consumer to publish fixed SvcParams for the service.  This can cause an outage or service degradation if the service makes a backward-incompatible configuration change.  Accordingly, zone owners SHOULD NOT publish SvcParams for a TargetName that they do not control, and service operators should take caution when making incompatible configuration changes.


# Security Considerations

This document specifies the use of TLSA as a property of each connection attempt.  In environments where DANE is optional, this means that the fallback procedure might use DANE for some connection attempts but not others.

This document only specifies the use of TLSA records when all relevant DNS records (including SVCB, TLSA, and CNAME records) were resolved securely.  If any of these resolutions were insecure (as defined in {{Section 4.3 of !RFC4035}}), the client MUST NOT rely on the TLSA record for connection security.  However, if the client would otherwise have used an insecure plaintext transport, it MAY use an insecure resolution result to achieve opportunistic security.


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
svc4.example.net.    HTTPS 0 xyz.cdn.example.
xyz.cdn.example.     A     192.0.2.1
~~~

The TLSA QNAME is `_443._tcp.xyz.cdn.example`.

## QUIC and CNAME

Given service URI `https://www.example.com` and records:

~~~
www.example.com.  CNAME api.example.com.
api.example.com.  HTTPS 1 svc4.example.net alpn=h2,h3 port=8443
svc4.example.net. CNAME xyz.cdn.example.
~~~

If the connection attempt is using HTTP/3, the transport label is set to `_quic`; otherwise `_tcp` is used.

The initial TLSA QNAME would be one of:

* `_8443._quic.xyz.cdn.example`
* `_8443._tcp.xyz.cdn.example`

If no TLSA record is found, the fallback TLSA QNAME would be one of:

* `_8443._quic.svc4.example.net`
* `_8443._tcp.svc4.example.net`

## DNS ServiceMode

Given a DNS server `dns.example.com` and record:

~~~
_dns.dns.example.com. SVCB 1 dns.my-dns-host.example. alpn=dot
~~~

The TLSA QNAME is `_853._tcp.dns.my-dns-host.example`.  The port and protocol are inferred from the "dot" ALPN value.

## DNS AliasMode

Given a DNS server `dns.example.com` and records:

~~~
_dns.dns.example.com.     SVCB 0 dns.my-dns-host.example.
dns.my-dns-host.example.  SVCB 1 . alpn=doq
~~~

The TLSA QNAME is `_853._quic.dns.my-dns-host.example`.  The port and protocol are inferred from the "doq" ALPN value.

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


# IANA Considerations

IANA is instructed to add the following entry to the "Underscored and Globally Scoped DNS Node Names" registry ({{?RFC8552, Section 4}}):

| RR Type | _NODE NAME | Reference       |
| ------- | ---------- | --------------- |
| TLSA    | _quic      | (This document) |

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
