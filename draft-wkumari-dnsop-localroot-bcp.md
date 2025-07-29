---
title: "Making Local-Rool a Best Current Practice"
#abbrev: "TODO - Abbreviation"
category: bcp

docname: draft-wkumari-dnsop-localroot-bcp-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
#number:
#date:
consensus: true
v: 3
area: "Operations and Management"
workgroup: "Domain Name System Operations"
keyword:
 - DNS
 - DNSSEC
 - zone cut
 - delegation
 - referral
updates: RFC8806
venue:
  group: "Domain Name System Operations"
  type: "Working Group"
  mail: "dnsop@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/dnsop/"
#  github: "ableyjoe/draft-jabley-dnsop-zone-cut-to-nowhere"
#  latest: "https://ableyjoe.github.io/draft-jabley-dnsop-zone-cut-to-nowhere/draft-jabley-dnsop-zone-cut-to-nowhere.html"

author:
 -
    fullname: Warren Kumari
    organization: Google, Inc.
    email: warren@kumari.net
 -
    fullname: Wes Hardaker
    organization: USC/ISI
    email: ietf@hardakers.net
# TODO: Jim Reid, Geoff.


normative:
  RFC8806:
  RFC8976:  # Zone Checksum

informative:
  RFC5936:  # DNS Zone Transfer

  BIND-MIRROR: https://bind9.readthedocs.io/en/stable/reference.html#namedconf-statement-type%20mirror
  UNBOUND-AUTH-ZONE: https://nlnetlabs.nl/documentation/unbound
  KNOT-PREFILL: https://knot-resolver.readthedocs.io/en/stable/modules-prefill.html

# TODO:

# DONE - 1: Add Zone Checksum
# DONE - 2: Look up BIND and Unbound support.
# 3: This recommends ("Operation Considerations") using HTTP. Is this a good idea? Need to discuss the bootstrapping issue, and load balancing.
# 4: Security Considerations - benefits, and any other issues.


--- abstract

RFC 8806 (colloqually known as "LocalRoot") defines a mechanism whereby a
recursive resolver can fetch the contents of an entire zone, and place this
information into the resolver's cache.

This has a number of benefits, including increasing reliability, increasing
performance, improving privacy, and decreasing or mitigating the effect of some
types of DoS attacks.

While RFC 8806 is natively supported by the majority of DNS resolver
implementations, it remains tricky to configure and maintain. This document
recommends that DNS resolver software simplify this configuration, and further
suggests that it become the default.

This document updates Section 2 of RFC8806 by relaxing the requirement that
implementations MUST run an authorative service.

/* Ed (WK): I started writing this as rfc8806-bis, but as I did so I realized
that it is likely better as a standalone docment. */


--- middle

# Introduction

{{!RFC8806}} provides "a method for the operator of a recursive
resolver to have a complete root zone locally, and to hide queries
for the root zone from outsiders.  The basic idea is to create an up-
to-date root zone service on the same host as the recursive server,
and use that service when the recursive resolver looks up root
information."

While {{!RFC8806}} behavior can be achieved by "manually" configuring software
the act as a secondary server for the root-zone (see {{!RFC8806}} Section
B.1. Example Configuration: BIND 9.12 and Section B.2 Example Configuration:
Unbound 1.8), most resolver implmentations now support simpler, and more
robust, configuration mechanisms to enable this support. For example, ISC BIND
9.14 and above supports "mirror" zones, Unbound 1.9 supports "auth-zone", and
Knot Resolver uses its "prefill" module to load the root zone information. See
Appendix A for more detail. In additon to providing simpler configuration of
the LocalRoot mechanism, these mechanisms suport "falling back" to querying
the root-servers directly if they are unable to fetch the entire root zone.


# Conventions and Definitions {#definitions}

{::boilerplate bcp14-tagged}

# Making RFC8806 behavior be a Best Current Practice

{{!RFC8806}} is an Informational document, which described a mechanism which
resolver operators could choose to use.

This document:

1. promotes the behavior in {{!RFC8806}} to be a Best Current Practice.
2. RECOMMENDS that resolver implementations provide a simple configuration
   option to enable or disable functionality and
3. RECOMMENDS that resolver implementations enable this behavior by default and
4. RECOMMENDS that {{!RFC8976}} be used to validate the zone information
   before loading it.

# Changes from RFC8806

{{!RFC8806}} Section 2 (Requirements) stated that:
> "o  The system MUST be able to run an authoritative service for the
      root zone on the same host.  The authoritative root service MUST
      only respond to queries from the same host.  One way to assure not
      responding to queries from other hosts is to run an authoritative
      server for the root that responds only on one of the loopback
      addresses (that is, an address in the range 127/8 for IPv4 or ::1
      in IPv6).  Another method is to have the resolver software also
      act as an authoritative server for the root zone, but only for
      answering queries from itself."

This document relaxes this requirement. Some resolver implementations achieve
the behavior described in RFC8806 by fetching the zone information and "prefilling" their cache with this information. As the resulting behavior is (largely) indistinguishable from the mechanism defined in RFC8806, this is viewed as simply being an acceptable implementation decision.

/* Ed (WK): Y'all know what I mean, but this could be worded better! How an implementation decides to implement LocalRoot behavior shouldn't matter, just that they behave.... */


# Applicability

This behavior should apply to all general purpose recursive resolvers.

# Operational Considerations

In order for the {{!RFC8806}} mechanism to be effective, a resolver must be
able to fetch the contents of the entire root zone. This is currently usually
performed through AXFR ({{!RFC5936}}), although some resolvers may allow fetching this information via HTTP. In order for AXFR to work, the resolver
must be able to use TCP (which is already required by {{!RFC7766}}).

If and where possible, HTTP should be preferred as it will allow for compression, and the possibility of using low cost, well distrubuted CDNs to distrubute the zone files.

Resolver software MUST validate the contents of the zone before using it. This
SHOULD be done using the mechanism in {{!RFC8976}}, but MAY be done by simply validating every signed record in a zone with DNSSEC [RFC4033].

/* Ed (WK): We might want to add some more discussions around failure handling, but, 1:  {{!RFC8806}} already covers much of this and 2: "don't teach your grandmother to suck eggs" - implementations already handle this, so let's not try to overspecify or overconstrain what they do.

# Security Considerations

/* Ed (WK): Fill this in. I think that it just contains descriptions of the benefits from RFC8806, but I'm guessing that there are some other concerns too... */


# IANA Considerations

This document has no IANA actions.


--- back


# Acknowledgments
{:numbered="false"}

The authors have discussed this idea with many people, and have likely
forgotten to acknowledge and credit many of them. If we discussed this with you,
and you are not listed, please please let us know and we'll add you.

The authors would like to thank Vint Cerf, John Crain, Puneet Sood, Robert Story, Suzanne Woolf.

In addition, one of the authors would like to once again thank the bands "Infected Mushroom", "Kraftwerk", and "deadmau5" for providing the soundtrack
to which this was written.

# Appendix A: Example Configurations
{:numbered="false"}

These examples are provided to show how the LocalRoot mechanism can be configured in various resolver implementations. They are not intended to be exhaustive, and may not work with all versions of the software.

/* Ed (WK): These examples are just to get started. We would appreciate contributions from the resolver operators. */

/* Ed (WK): Yes, we are fully aware of the circular dependency of trying to
resolve e.g www.internic.net when bootstrapping. More discussion on serving
the root zone over HTTP by IP will be added later. */

## ISC BIND 9.14 and above
See the BIND documentation for [mirror zones](https://bind9.readthedocs.io/en/stable/reference.html#namedconf-statement-type%20mirror).


Example configuration using a "mirror" zone:

```text
zone "." {
    type mirror;
};
```

## Knot Resolver
See the Knot Resolver [Cache prefilling](https://knot-resolver.readthedocs.io/en/v5.0.1/modules-prefill.html?highlight=cache%20prefilling) documentation for more information.

The following example configuration will prefill the root zone using HTTPS:

```text
modules.load('prefill')
prefill.config({
      ['.'] = {
              url = 'https://www.internic.net/domain/root.zone',
              interval = 86400  -- seconds
              ca_file = '/etc/pki/tls/certs/ca-bundle.crt', -- optional
      }
})
```

## Unbound 1.9 and above
See the Unbound documentation for [Authority Zone Options](https://unbound.docs.nlnetlabs.nl/en/latest/manpages/unbound.conf.html#unbound-conf-auth-url) configuration.

The following example configuration will prefill the root zone using HTTPS:

```text
auth-zone:
  name: "."
  url: "https://www.internic.net/domain/root.zone"
  zonefile: "root.zone"
        fallback-enabled: yes
    for-downstream: no
    for-upstream: yes
    zonefile: "root.zone"
  prefetch: yes
```



