---
title: Extensible Delegation for DNS
abbrev: DELEG
docname: draft-dnsop-deleg-latest
date: {DATE}
category: std

ipr: trust200902
workgroup: dnsop
area: Internet
category: info
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
-
    ins: T. April
    name: Tim April
    organization:
    email: ietf@tapril.net
-
    ins: P.Špaček
    name: Petr Špaček
    organization: ISC
    email: pspacek@isc.org
-
    ins: R.Weber
    name: Ralf Weber
    organization: Akamai Technologies
    email: rweber@akamai.com


--- abstract

Within the DNS, there is no way to delegate child domain to use newer or modern methods or transportation protocol. This draft proposes a new extensible DNS record type DELEG to allow delegation of a domain with additional capabilities.

--- middle

# Introduction

Resolvers currently rely on the NS records in the parent and child zones to provide and confirm the nameservers that are authoritative for each zone. These are not extensible so any new feature with regards to delegation requires additional records like e.g was done with the introduction the DS record for DNSSEC.

The DELEG records uses the SVCB record format defined in {{?I-D.draft-ietf-dnsop-svcb-https-00}}, using a subset of the already defined service parameters as well as new parameters described in this document.

## Reasoning for changing delegation

* The current DNS protocol does not allow secure upgrade to new transport mechanisms between auth and resolver - e.g. authenticated DNS-over-TLS from resolver to auth.
* The current delegation with NS and glue records requires a leap of faith because these records are not signed. Spoofed delegation can be detected eventually if the domain is signed, but only after traffic is (potentially) sent to the attacker’s endpoint.
* There	is no secure way to signal in delegation what capabilities authoritative servers support. The client cannot rely on any new behavior and currently must resort to trial-and-error with EDNS which is unsigned.
* DNS operators don’t have a formal role in the current system. Consequently, registries/registrars generally don’t talk to operators and users must act as go-between and make technical changes they don’t understand.
    * Asking domain owners to add new NS records, modify DS records when rolling keys etc. This is very inflexible when an operator needs to make technical changes.


## Introductory Examples

To introduce DELEG record, this example shows a possible response from an authoritative in the authority section of the DNS response when delegating to another nameserver.


    example.com.  86400  IN DELEG  1 config1.example.com. ( transports=dot,
                    dnsTlsFingerprints=["MIIS987SSLKJ...123===",
                        "MII3SODKSLKJ...456==="],
                    ipv4hint=34.23.34.54,24.45.23.78,
                    ipv6hint=2a33:2423::3,2400:DEAD:BEEF::53 )
    example.com.  86400  IN NS     ns1.example.com.
    ns1.example.com.    86400   IN  NS  192.0.2.1

In this example, the authoritative nameserver is delegating using DNS-over-TLS
Like in SVCB, DELEG also offer the ability to use the Alias form delegation. The example below shows an example where example.com is being delegated with a DELEG AliasForm record which can then be further resolved using standard SVCB to locate the actual parameters.

    example.com.  86400  IN DELEG 0   config2.example.net.
    example.com.  86400  IN NS     DELEG.example.net.

The example.net authoritative server may return the following SVCB records in response to a query as directed by the above records.

    config2.example.net 3600    IN SVCB . ( transports=dot,
                    dnsTlsFingerprints=["MIIS987SSLKJ...123==="],
                    ipv4hint=34.23.34.54,24.45.23.78,
                    ipv6hint=2a33:2423::3,
                    ds=12345,13,1,DSFDSAFDSAFVCFSGFDSGDSAGFSA )

The above records indicate to the client that the actual configuration for the example.com zone can be found at config2.example.net

Later sections of this document will go into more detail on the resolution process using these records.

## Goal of the DELEG record

The primary goal of the DELEG records is to provide zone owners a way to signal to clients how to connect and validate a child domain that can coexist with NS records in the same zone, and do not break software that does not support them.

The DELEG record is authortiative in the parent zone and if signed has to be signed with the key of the parent zone. The target of an alias record is an SVCB record that exists and can be signed in the zone it is pointed at, including the child zone.

## Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in \\
BCP 14 {{?RFC2119}} {{?RFC8174}} when, and only when, they appear in
all capitals, as shown here.

# DELEG Record Type

The SVCB record allows for two types of records, the AliasForm and the ServiceForm. The DELEG record takes advantage of both and each will be described below in depth.

## Difference between the records

This document uses two different resource record types. Both records have the same functionality, with the difference between them being that the DELEG record MUST only be used at a delegation point while the SVCB, is used as a normal resource record does not indicate that the label is being delegated. For example, take the following DELEG record:

    example.com.  86400  IN DELEG 1   config2.example.net. ( transports=doq )

When a client receives the above record, the resolver should send queries for any name under example.com to the nameserver at config2.example.com unless further delegated. By contrast, when presented with the records below:

    example.com.  86400  IN DELEG 0   config3.example.org.
    config3.example.org.  86400  IN SVCB 1 . ( transports=dot )

A resolver trying to resolve a name under example.com would get the first record above from the parent authoritative server, .COM, indicating that the SVCB records found at config3.example.org should be used to locate the authoritative nameservers of example.com, and other parameters.

The primary difference between the two records is that the DELEG record means that anything under the record label should be queried at the delegated server while the SVCB record is just for redirection purposes, and any names under the record's label should still be resolved using the same server unless otherwise delegated.

It should be noted that both DELEG and SVCB records may exist for the same label, but they will be in differen zones Below is an example of this:

    Zone example.com.:
    example.com.  86400  IN DELEG 0   c1.example.org.

    Zone .example.org.:
    c1.example.org.  86400  IN DELEG  1   config3.example.net. ( transports=dot )

    Zone c1.example.org
    c1.example.org.  86400  IN SVCB 1   config2.example.net. ( transports=dot )
    c1.example.org.  86400  IN NS test.c1.example.org.
    test.c1.example.org. 600 IN A 192.0.2.1

In the above case, the DELEG record for c1.example.org would only be used when trying to resolve names at or below c1.example.org. This is why when an AliasForm DELEG or SVCB record is encountered, the resolver MUST query for the SVCB record associated with the given name.


## AliasForm Record Type

In order to take full advantage of the AliasForm of DELEG and SVCB, the parent, child and resolver must support these records. When supported, the use of the AliasForm will allow zone owners to delegate their zones to another operator with a single record in the parent. AliasForm SVCB records SHOULD appear in the child zone when used in the parent. If a resolver were to encounter an AliasForm DELEG or SVCB record, it would then resolve the name in the TargetName of the original record using SVCB RR type to receive the either another AliasForm record or a ServiceForm SVCB record.

For example, if the name www.example.com was being resolved, the .com zone may issue a referral by returning the following record:

    example.com.    86400    IN  DELEG     0   config1.example.net.

The above record would indicate to the resolver that in order to obtain the authoritative nameserver records for example.com, the resolver should resolve the RR type SVCB for the name config1.example.net.

### Multiple Service Providers

Some zone owners may wish to use multiple providers to serve their zone, in which case multiple DELEG AliasForm records can be used. In the event that multiple DELEG AliasForm records are encountered, the resolver SHOULD treat those as a union the same way this is done with NS records, picking one at random for the first lookup and eventually discovering the others. How exactly DNS questions are directed and split between configuration sets is implementation specific:

    example.com.    86400    IN  DELEG     0   config1.example.net.
    example.com.    86400    IN  DELEG     0   config1.example.org.

DRAFT NOTE: SVCB says that there "SHOULD only have a single RR". This ignores that but keep the randomization part. Section 2.4.2 of SVCB

### Loop Prevention

Special care should be taken by both the zone owner and the delegated zone operator to ensure that a lookup loop is not created by having two AliasForm records rely on each other to serve the zone. Doing so may result in a resolution loop, and likely a denial of service. Any clients implementing DELEG and SVCB MUST implement a per-resolution limit of how many AliasForm records may be traversed when looking up a delegation to prevent infinite looping. When a loop is detected, like with the handling of CNAME or NS, the server MUST respond to the client with SERVFAIL.

## ServiceForm Record Type

The ServiceForm of the DELEG and SVCB records are likely to be the more popular of the two. They work the same way as the SVCB or HTTPSSVC records do by providing a priority and list of parameters associated with the TargetName. In addition to being able to identify which protocols are supported by the authoriatative server, the ServiceForm record will also allow providers to operate different protocols on different addresses.

### SvcFieldPriority

As defined in the DNS SVCB document {{?I-D.draft-ietf-dnsop-svcb-httpssvc-00}}, the SvcFieldPriority values SHOULD be used to dictate the priority when multiple DELEG or SVCB records are returned.

### TargetName

As defined in the SVCB document {{?I-D.draft-ietf-dnsop-svcb-httpssvc-00}}, the TargetName provides the hostname to which the DELEG or SVCB record refers. The TargetName field MUST be set and MUST NOT be "." for DELEG records. Records with a TargetName of "." SHOULD be discarded.

^^^ Does that matter when IP hints are provided? I would think we don't need to name servers if we don't feel like it. (Of course TLS might require a name, but that's different story than DELEG itself.)

DRAFT NOTE: Should u-label and a-label be expanded? I'm leaning towards not expanding.

### SvcParamKeys

The following DNS SVCB parameters are defined for the DELEG and SVCB ServiceForms.

#### "transports"

The "transports" SvcParamKey defines the list of transports offered by the nameserver named in the TargetName.

The existing "alpn" SvcParamKey was not reused for DELEGdue to the restriction that the "alpn" SvcParamValues are limited to those defined in the TLS Application-Layer Protocol Negotiation (ALPN) Protocol IDs registry. Plaintext DNS traffic is not, and should not be listed in that registry, but is required to support the transition to encrypted transport in DELEG and SVCB records.

Below is a list of the transport values defined in this document:

* "do53": indicates that a server supports plaintext, unencrypted DNS traffic over UDP or UDP as defined in {{?RFC1035}} and {{?RFC1034}} and the updates to those RFCs.
* "dot": indicates that the server supports encrypted DNS traffic over DNS-over-TLS as defined in {{?RFC7858}}.
* "doh": indicates that the server supports encrypted DNS traffic over DNS-over-HTTPS as defined in {{?RFC8484}}. Records that use the DoH service form may be further redirected with HTTPSSVC records in the delegated zone.
* "doq": indicates that the server supports encrypted DNS traffic via DNS over QUIC Connections as defined in {{?RFC9250}}

The order of the keys in the list dictate the order which the nameserver SHOULD be contacted in. The client SHOULD compare the order of available transports with the set of transports it supports to determine how to contact the selected nameserver.

The presentation format of the SvcParamValue is a comma delimited quoted string of the available transport names. The wire format for the SvcParamValue is a string of 16-bit integers representing the TransportKey values as described in the "NS2/NS2T Transport Parameter Registry".

#### "dnsTlsFingerprints"

The "dnsTlsFingerprints" SvcParamKey is a list of fingerprints for the TLS certificates that may be presented by the nameserver. This record SHOULD match the TLSA record as described in {{?RFC6698}}. Due to bootstrapping concerns, this SvcParamKey has been added to the DELEG record as the TLSA records would only be resolveable after the initial connetion to the delegated nameserver was established. When this field is not present, certificate validation should be performed by either DANE or by traditional TLS certification validation using trusted root certification authorities.

The presentation and wire format of the SvcParamValue is the same as the presentation and wire format described for the TLSA record as defined in {{?RFC6698}}, sections 2.1 and 2.2 respectively.

## Deployment Considerations

The DELEG and SVCB records intends to replace the NS record while also adding additional functionality in order to support additional transports for the DNS. Below are discussions of considerations for deployment.

### AliasForm and ServiceForm in the Parent

Both the AliasForm and ServiceForm records MUST NOT be returned for the DELEG record.
^^^ Why? I think ability to combine in-bailwick and out-of-bailwing NS names is normal thing and does not cause more trouble.


### Rollout

When introduced, the DELEG and SVCB record will likely not be supported by the Root or second level operators, and may not be for some time. In the interim, zone owners may place these records into their zones. However this is only useful for further delegations down the tree as an SVCB record at the zone apex alone does not indicate a new delegation type. The only way to discover new delegations is with the DELEG record at the parent.


DRAFT NOTE: Should we include something here that an authoritative MAY include DELEG records in the additionals section of responses to encourage resolvers to upgrade? What about an EDNS0 option from the authority to signal that it is capable of alternat transports?

### Availability

If a zone operator removes all NS records before DELEG and SVCB records are implemented by all clients, the availability of their zones will be impacted for the clients that are using non-supporting resolvers. In some cases, this may be a desired quality, but should be noted by zone owners and operators.

## Response Size Considerations

For latency-conscious zones, the overall packet size of the delegation records from a parent zone to child zone should be taken into account when configuring the NS, DELEG and SVCB records. Resolvers that wish to receive DELEG and SVCB records in response SHOULD advertise and support a buffer size that is as large as possible, to allow the authoritative server to respond without truncating whenever possible.


# DNSSEC and DELEG

Unlike NS records DELEG records are always signed by the parent similar to DS records.
SVCB records the DELEG record point to, are signed as normal record types in a signed child/leaf zone.

# Privact Considerations

All of the information handled or transmitted by this protocol is public information published in the DNS.

# Security Considerations

TODO: Fill this section out

## Availability of zones without NS

## Resolution procedure

TODO: full resolution using QNAME min asking for DELEG

### Connetion Failures

When a resolver attempts to access nameserver delegated by a DELEG or SVCB record, if a connection error occurs, such as a certificate mismatch or unreachable server, the resolver SHOULD attempt to connect to the other nameservers delegated to until either exhausting the list or the resolver's policy indicates that they should treat the resoltion as failed.

The failure action when failing to resolve a name with DELEG/SVCB due to connection erros is dependant on the resolver operators policies. For resolvers which strongly favor privacy, the operators may wish to return a SERVFAIL when the DELEG/SVCB resoltion process completes without successfully contacting a delegated nameserver(s) while opportunisitic privacy resolvers may wish to attempt resolution using any NS records that may be present.

# IANA Considerations

## New registry for DELEG transports

The "DELEG Transport Parameter Registry" defines the namespace for parameters, including string representations and numeric values. This registry applies to the "transports" DNS SVCB format, currently impacting the DELEG RR Type.

ACTION: create and include a reference to this registry.

### Procedure

A registration MUST include the following fields:

* Name: The transport type key name
* TransportKey: A numeric identifier (range 0-65535)

* Meaning: a short description
* Protocol Specification
* Pointer to specification text

### Initial Contents

The "DELEG/SVCB Transport Parameter Registry" shall initially be populated with the registrations below:

| TransportKey | Name | Meaning | Protocol Specification | Reference |
|--------------|------|---------|------------------------|-----------|
| 0            | key0 | Reserved| Reserved              | (This Document) |
| 1            | do53 | Unencrypted, Plaintext DNS over UDP or TCP | RFC1035 | (This Document) |
| 2            | dot | DNS-over-TLS | RFC7858             | (This Document) |
| 3            | doh | DNS-over-HTTPS | RFC8484            | (This Document) |
| 4            | doq | DNS over Dedicated QUIC Connections | {{?RFC9250}} |
| 65280-65534  | keyNNNNN | Private Use | Private Use            | (This Document) |
| 65535        | key65535 | Reserved| Reserved              | (This Document) |

## New SvcParamKey Values

This document defines new SvcParamKey values in the "Service Binding (SVCB) Parameter Registry".

| SvcParamKey | NAME | Meaning | Reference |
|-------------|------|---------|-----------|
| TBD1        | transports | | (This Document) |
| TBD2        | dnsTlsFingerprints | | (This Document) |

--- back

# Acknowledgements

This draft is heavily based on past work (draft-tapril-ns2) done by Tim April and thus extends the thanks to the people helping on this which are:
John Levine, Erik Nygren, Jon Reed, Ben Kaduk, Mashooq Muhaimen, Jason Moreau, Jerrod Wiesman, Billy Tiemann, Gordon Marx and Brian Wellington.

# TODO

RFC EDITOR:
: PLEASE REMOVE THE THIS SECTION PRIOR TO PUBLICATION.

* Write a security considerations section
* Add the dohpath references RFC
* Define and give examples for DNSSEC svcparams
* worked out resolution example including alias form delegation
* DoH URI teamplte does not include post


# Change Log

RFC EDITOR:
: PLEASE REMOVE THE THIS SECTION PRIOR TO PUBLICATION.

~~~
01234567890123456789012345678901234567890123456789012345678901234567891