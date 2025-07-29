---
title: "Making LocalRoot a Best Current Practice"
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
  github: "https://github.com/wkumari/draft-wkumari-dnsop-localroot-bcp"
  latest: "https://wkumari.github.io/draft-wkumari-dnsop-localroot-bcp/draft-wkumari-dnsop-localroot-bcp.html"

author:
 -
    fullname: Warren Kumari
    organization: Google, Inc.
    email: warren@kumari.net
 -
    fullname: Wes Hardaker
    organization: USC/ISI
    email: ietf@hardakers.net
 -
    ins: J. Reid
    name: Jim Reid
    org: RTFM llp
    street: St Andrews House
    city: 382 Hillington Road, Glasgow Scotland
    code: G51 4BL
    country: UK
    email: jim@rfc1035.com
 -
    ins: G. Huston
    fullname: Geoff Huston
    organization: APNIC
    email: gih@apnic.net
    street: 6 Cordelia St
    city: South Brisbane
    code: QLD 4101
    country: Australia

normative:
  RFC8806:
  RFC8976:
  RFC9364:  # DNSSEC

informative:
  RFC5936:  # DNS Zone Transfer
  RFC7766:  # DNS Transport over TCP

  BIND-MIRROR:
    title: "BIND 9 Mirror Zones"
    target: https://bind9.readthedocs.io/en/stable/reference.html#namedconf-statement-type%20mirror
  UNBOUND-AUTH-ZONE:
    title: "Unbound Auth Zone"
    target: https://nlnetlabs.nl/documentation/unbound
  KNOT-PREFILL:
    title: "Knot Resolver Prefill"
    target: https://knot-resolver.readthedocs.io/en/stable/modules-prefill.html


--- abstract

RFC 8806 (often called "LocalRoot") defines a mechanism whereby a
recursive resolver can fetch the contents of an entire zone and place this
information into the resolver's cache.

This has several benefits, including increasing reliability, increasing
performance, improving privacy, and decreasing or mitigating the effect of some
types of DoS attacks.

While the majority of DNS resolver implementations natively support RFC 8806,
it remains tricky to configure and maintain. This document recommends that DNS
resolver software simplify this configuration, and further suggests that it
become the default.

This document updates Section 2 of RFC8806 by relaxing the requirement that
implementations MUST run an authoritative service.


/* Ed (WK): Open questions / ToDo / Notes (to be removed before publication):

1. I started writing this as rfc8806-bis, but as I did so I realized
that it is likely better as a standalone document.

1. DONE - Add Zone Checksum

2. DONE - Look up BIND and Unbound support.

3. This document recommends ("Operation Considerations") using HTTP. Need to discuss the bootstrapping issue, and load balancing.

4. Security Considerations - flesh this out. I think that it just contains descriptions of the benefits from RFC8806, but I suspect there will be some other concerns too.

*/

--- middle

# Introduction

{{RFC8806}} provides "a method for the operator of a recursive
resolver to have a complete root zone locally, and to hide queries
for the root zone from outsiders.  The basic idea is to create an up-to-date
root zone service on the same host as the recursive server,
and use that service when the recursive resolver looks up root
information."

While {{RFC8806}} behavior can be achieved by "manually" configuring software
the act as a secondary server for the root-zone (see {{RFC8806}} Section
B.1. Example Configuration: BIND 9.12 and Section B.2 Example Configuration:
Unbound 1.8), most resolver implmentations now support simpler, and more
robust, configuration mechanisms to enable this support. For example, ISC BIND
9.14 and above supports "mirror" zones, Unbound 1.9 supports "auth-zone", and
Knot Resolver uses its "prefill" module to load the root zone information. See
Appendix A for details. In addition to providing simpler configuration of
the LocalRoot mechanism, these mechanisms support "falling back" to querying
the root-servers directly if they are unable to fetch the entire root zone.


# Conventions and Definitions {#definitions}

{::boilerplate bcp14-tagged}

# Making RFC8806 behavior be a Best Current Practice

{{RFC8806}} is an Informational document, which describes a mechanism that
resolver operators can use to improve the performance, reliability, and privacy
of their resolvers.

This document:

1. promotes the behavior in {{RFC8806}} to be a Best Current Practice.
2. RECOMMENDS that resolver implementations provide a simple configuration
   option to enable or disable functionality and
3. RECOMMENDS that resolver implementations enable this behavior by default and
4. RECOMMENDS that {{RFC8976}} be used to validate the zone information
   before loading it.

# Changes from RFC8806

{{RFC8806}} Section 2 (Requirements) states that:

  > The system MUST be able to run an authoritative service for the
  > root zone on the same host.  The authoritative root service MUST
  > only respond to queries from the same host.  One way to assure not
  > responding to queries from other hosts is to run an authoritative
  > server for the root that responds only on one of the loopback
  > addresses (that is, an address in the range 127/8 for IPv4 or ::1
  > in IPv6).  Another method is to have the resolver software also
  > act as an authoritative server for the root zone, but only for
  > answering queries from itself.

This document relaxes this requirement. Some resolver implementations achieve
the behavior described in RFC8806 by fetching the zone information and "prefilling" their cache with this information. As the resulting behavior is (essentially) indistinguishable from the mechanism defined in RFC8806, this is viewed as simply being an acceptable implementation decision.

/* Ed (WK): Y'all know what I mean, but this could be worded better! How an implementation decides to implement LocalRoot behavior should not matter, just that they behave.... */


# Applicability

This behavior should apply to all general-purpose recursive resolvers.

# Operational Considerations

In order for the {{RFC8806}} mechanism to be effective, a resolver must be
able to fetch the contents of the entire root zone. This is currently usually
performed through AXFR ({{RFC5936}}), although some resolvers may allow fetching this information via HTTP. In order for AXFR to work, the resolver
must be able to use TCP (which is already required by {{RFC7766}}).

Where possible, HTTP should be preferred as it will allow for compression, and the possibility of using low-cost, well-distributed CDNs to distribute the zone files.

Resolver software MUST validate the contents of the zone before using it. This
SHOULD be done using the mechanism in {{RFC8976}}, but MAY be done by simply validating every signed record in a zone with DNSSEC {{RFC9364}}.

/* Ed (WK): We might want to add some more discussions around failure handling, but, 1:  {{RFC8806}} already covers much of this and 2: "don't teach your grandmother to suck eggs" - implementations already handle this, so let's not try to overspecify or overconstrain what they do.

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

/* Ed (WK): These examples are just to get started. We would appreciate contributions from the resolver operators.

Yes, we are fully aware of the circular dependency of trying to
resolve e.g www.internic.net when bootstrapping. More discussion on serving
the root zone over HTTP by IP will be added later. */

## ISC BIND 9.14 and above
{:numbered="false"}

See the BIND documentation for [mirror zones](https://bind9.readthedocs.io/en/stable/reference.html#namedconf-statement-type%20mirror).


Example configuration using a "mirror" zone:

```text
zone "." {
    type mirror;
};
```

## Knot Resolver
{:numbered="false"}

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
{:numbered="false"}

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



