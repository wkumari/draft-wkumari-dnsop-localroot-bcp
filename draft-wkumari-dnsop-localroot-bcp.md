---
title: "Running a Root Server Local to a Resolver"
#abbrev: "TODO - Abbreviation"
category: bcp

docname: draft-wkumari-dnsop-localroot-bcp-latest
submissiontype: IETF
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
    organization: USC/ISI and Google, Inc.
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
  -
    fullname: David Conrad
    organization: Layer 9 Technologies, LLC.
    email: david.conrad@layer9.tech

normative:
  BCP237:
  RFC1982:  # SOA math 
  RFC4033:  # DNSSEC
  RFC8198:
  RFC8499:
  RFC8806:
  RFC8976:
  RFC9110:  # HTTP methods

informative:
  RFC5936:  # DNS Zone Transfer
  RFC7766:  # DNS Transport over TCP
  RFC9156:  # QNAME Minimisation
  RFC9110:  # HTTP Semantics

  BIND-MIRROR:
    title: "BIND 9 Mirror Zones"
    target: https://bind9.readthedocs.io/en/stable/reference.html#namedconf-statement-type%20mirror
  UNBOUND-AUTH-ZONE:
    title: "Unbound Auth Zone"
    target: https://nlnetlabs.nl/documentation/unbound
  KNOT-PREFILL:
    title: "Knot Resolver Prefill"
    target: https://knot-resolver.readthedocs.io/en/stable/modules-prefill.html
  QNAMEMIN:
    title: DNS Query Privacy
    target: https://www.potaroo.net/ispcol/2019-08/qmin.html
  LOCALROOTPRIVACY:
    title: Analyzing and mitigating privacy with the DNS root service
    target: http://ant.isi.edu/~hardaker/papers/2018-02-ndss-analyzing-root-privacy.pdf
  CACHEME:
    title: "Cache Me If You Can: Effects of DNS Time-to-Live"
    target: https://ant.isi.edu/~johnh/PAPERS/Moura19b.pdf
  XFRSCHEME:
    title: The DNS XFR URI Schemes
    target: draft-hardaker-dnsop-dns-xfr-scheme

--- abstract

Many DNS recursive resolver operators wish to prevent snooping by
third parties of requests sent to DNS root servers.  In addition, some
DNS recursive resolvers have longer-than-desired round-trip times to
the closest DNS root server; those resolvers may have difficulty
getting responses from the root servers, such as during a network
attack.  Resolvers can solve both of these issues by serving or
pre-caching a copy of the full root zone on the same server or within
the resolver
software.

This document shows how to fetch, cache and maintain such a copy of the root
zone, how to detect if it becomes stale, and mechanisms for handling
error states.  This specification is designed to increase the
resiliency, privacy and efficiency of DNS resolver services.

This document obsoletes RFC 8806.

/* Ed (WK): Open questions / ToDo / Notes (to be removed before publication):

1. I started writing this as rfc8806-bis, but as I did so I realized that it is
likely better as a standalone document.

1. This document recommends ("Operation Considerations") using HTTP(S) for
fetching the zone. We still need to add text to cover priming and discuss the
bootstrapping issue. In addition, we need to add text about loadbalancing and
fetching from multiple sources. Much of the premise behind RFC8806 is that it
doesn't matter where you fetch the zone from, as long as you validate it, and
use zone checksums {{RFC8976}}.
*/

--- middle

# Introduction

DNS recursive resolvers have to provide answers to all queries from
their clients, even those for domain names that do not exist.  For
each queried name that is within a top-level domain (TLD) that is not
in the recursive resolver's cache, the resolver must send a query to
a root server to get the information for that TLD or to find out that
the TLD does not exist.  Research shows that the vast majority of
queries going to the root are for names that do not exist in the root
zone.

Many of the queries from recursive resolvers to root servers get
answers that are referrals to other servers.  Malicious third parties
might be able to observe that traffic on the network between the
recursive resolver and root servers.

------

Caching the root zone data locally, commonly referred to as running a
"LocalRoot" instance, provides a method for the operator of a
recursive resolver to have a complete copy of the IANA root zone
locally rather than sending requests for it to the Root Server System
(RSS).  This can be implemented using a number of different
implementation techniques, but the net effect is the same: few, if
any, queries are sent to the actual RSS.

Note that enabling LocalRoot functionality in a resolver will probably
have little effect on improving resolver speed to a stub resolver for
good queries under Top Level Domains (TLDs), as the TTL for most TLDs
is already long-lived (two days in the current root zone). Thus the
data is typically already in a resolver's cache.  Negative answers
from the root servers are also cached in a similar fashion, though
potentially for a shorter time based on the SOA negative cache timing.

Two potential implementation mechanisms are documented herein for
achieving LocalRoot functionality: having the resolver pre-fetch the
root zone at regular intervals and pre-populate its cache with
information, or by running an authoritative server in parallel with
the recursive resolver that acts as a locally authoritative root
server.  To a client, the net effect of using any technique should be
nearly indistinguishable to that of a non-Localroot resolver.

A different approach to partially mitigating some of the problems that
a LocalRoot enabled resolver solves can be achieved using "Aggressive
Use of DNSSEC-Validated Cache" {{RFC8198}} functionality.

Readers are expected to be familiar with {{RFC8499}}.

# Conventions and Definitions {#definitions}

{::boilerplate bcp14-tagged}

# Making RFC8806 behavior be a Best Current Practice

Note: DNSOP needs to discuss whether to publish this as a BCP or as a
bis-document and making LocalRoot a proposed standard (RFC8806 is
informational)

{{RFC8806}} is an Informational document that describes a mechanism that
resolver operators can use to improve the performance, reliability, and privacy
of their resolvers.  This document concludes the experiment
{{RFC8806}} was a success.  The reality is that secure DNS resolution
using a local copy of the IANA root zone is possible because
technologies like DNSSEC and ZONEMD {{RFC8976}} allow for the contents
to be fetched from any location and subsequently verified and used
within validating resolvers.

This document:

1. promotes the behavior in {{RFC8806}} to be a Best Current Practice.
2. RECOMMENDS that resolver implementations provide a simple configuration
   option to enable or disable functionality, and
3. RECOMMENDS that resolver implementations enable this behavior by default. and
4. REQUIRES that {{RFC8976}} be used to validate the IANA root zone information
   before loading it.
5. Adds a mechanism for priming the list of places for fetching root zone data.
6. Adds protocol steps for ensuring resolution stability and resiliency.

## Changes from RFC8806

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

This document relaxes this requirement. Resolver implementations can
achieve the desired behavior of directly serving the contents of the
root zone via multiple implementation choices, beyond those listed in
{{RFC8806}}.  This can include what is described in RFC8806, but this
document allows for implementations to select any mechanism for
fetching and re-distributing the contents of the root zone on their
resolver service addresses. For example, this can be done by simply
"prefilling" the resolver's cache with the contents of the root
zone. As the resulting behavior is (essentially) indistinguishable
from the mechanism defined in RFC8806, this is viewed as being an
acceptable implementation decision.  In the end, the fundamental
requirement is simply: resolvers MUST return the records from the root
zone without modification.



## Applicability

This behavior should apply to all general-purpose recursive resolvers used on
the public Internet.

# LocalRoot enabled resolver requirements

In order to implement the mechanism described in this document:

- The resolver system MUST be able to validate the contents of the root zone
  using ZONEMD {{RFC8976}}, which also requires supporting
  DNSSEC for verifying the root zone's ZONEMD record.

- The resolver system MUST have a configured DNSSEC trust anchor as an
  up-to-date copy of the public part of the Key Signing Key (KSK)
  {{RFC4033}} or used to sign the DNS root or its DS record.

- The resolver system MUST be able to retrieve a copy of the entire root zone
  (including all DNSSEC-related records) {{protocol-steps}}.

- The resolver system MUST be able to fall back to querying the
  authoritative RSS servers whenever the local copy of the root zone
  data is unavailable or has been deemed stale {{protocol-steps}}.

A corollary of the above list is that a resolver operating as a
LocalRoot MUST return equivalent answers about the DNS root or any
other part of the DNS as if it was not operating as a LocalRoot.

# Functionality of a LocalRoot enabled resolver

The functionality of LocalRoot enabled resolver includes:

1. Identifying locations from where root zone data can be obtained
   {{root-zone-sources}}.
2. Downloading and refreshing the root zone data from one of the
   publication points {{protocol-steps}}.
3. Integrating and serving the data while performing DNS resolutions
   {{integrating-root-zone-data}}


## Identifying locations from where root zone data can be obtained {{root-zone-sourecs}}

In order for the LocalRoot functionality to be effective, an
implementation must be able to fetch the contents of the entire IANA
root zone.  Implementations can find sources in a number of ways,
including but not limited to:

1. Using a locally configured list of sources from which to fetch a
   copy of the IANA root zone.
2. Using a list of sources distributed with the resolver software
   itself.
3. By downloading a copy of available sources from the IANA using the
   sources described in {{iana-root-zone-list}}.
   
To support LocalRoot implementations, IANA will aggregate, publish and
maintain a list of IANA DNS root zone sources at *TBD-URL*
{{iana-root-zone-list}}.  Guidance to IANA or for other entities
wishing to collect and redistribute a list of sources for IANA root
server data is discussed in RFCTBD.

### IANA maintained list of root zone publication points  {#iana-root-zone-list}

This list of IANA root zone data publication points available at
TBD-URL may be used when downloading and refreshing the root zone
data, as described in {{protocol-steps}}.  Specifically, this IANA DNS
root zone publication list MAY be used by the resolver software
directly, or by the operating system a resolver is deployed on, or by
a network operator when configuring a resolver.

The contents of the IANA DNS root publication points file MUST
verified as to its integrity as having come from IANA and MUST be
verified as complete.

The format of the IANA root zone data publication points file will
consist of two parts, separated by a line containing four dashes and a
newline ("----\\n").  The top section of the file contain a newline
delimited list of URLs {{?RFC2056}}.  The second section, following
the line containing four dashes, will contain a cryptographic checksum
or signature.  Note that the format of this file applies to the IANA
maintained list of root zone publication points, but may or may not be
a format used by other publication point aggregation lists.

URLs in the list may include any protocol capable of transferring DNS
zone data, including HTTPS {{RFC9110}}, AXFR
{{draft-hardaker-dnsop-dns-xfr-scheme}}, XoT
{{draft-hardaker-dnsop-dns-xfr-scheme}}, etc.

Any URLs that reference an unknown transfer protocol SHOULD be
discarded.  If after filtering the list there are no acceptable list
elements left, the resolver MUST revert to using regular DNS queries
to the IANA root zone instead of operating as a LocalRoot.

The first line of the cryptograhpic checksum section will contain a
checksum or signature type string specifying what the remaining lines
in the checksum or signature section will contain.

An minimal example publication point file, containing only a single
AXFR publication point of b.root-servers.net:

~~~~
axfr:b.root-servers.net/.
----
SHA256
67d687eb21e59321dbb8115c51d1b4ddbd6634362859d130ed77b47a4410656c
~~~~

#### Publication point operational considerations

Implementations SHOULD optimize retrieval to minimize impacts on the
server.  Because the list is not expected to change frequently,
implementations SHOULD refrain from querying the IANA source more than
once a week.

## Protocol steps {#protocol-steps}

When initializing an implementation's LocalRoot mechanism, the following
steps MAY be used to implement the LocalRoot functionality.  Note that
as long as the desired effect of performing normal DNS resolution
remains stable when combined with LocalRoot functionality, other
implementations MAY be used.

If local root zone data is unavailable at any point in these steps,
resolvers SHOULD fall back to performing DNS resolution by issuing
queries to the RSS as needed.  If a resolver is unable to do so, it
MUST respond to client requests with a SERVFAIL response code.


1. The resolver SHOULD use a list of root zone sources identified in
   {{root-zone-sources}} for obtaining a copy of the IANA root zone.

2. The resolver MUST select one of the available sources from step 1,
   and from it retrieve a current copy of the IANA root zone.
   Resolvers SHOULD prioritize sources they can fetch over from most
   efficiently.  When supported, HTTPS sources should be preferred as
   it allows for compression negotiation as well as the possibility of
   using low-cost, well-distributed CDNs to distribute the zone files.
   When querying a source of IANA root zone data, the resolver SHOULD
   minimize impact to the source by querying at a rate specified by
   the SOA refresh timer and SHOULD use data freshness checks such as
   the HEAD method {{RFC9110}} when using HTTP(s) or by querying the
   root zone's SOA over DNS first when using AXFR, IXFR or XoT.  Once
   fetched, an implementation MUST NOT make use of an obtained IANA
   root zone with a SOA serial number less than any previously
   obtained copy {{RFC1982}}.

3. If the resolver failed to retrieve the IANA root zone content in
   step 2, or the zone content's serial number was deemed to be older
   than an already cached copy, and there are other available sources
   from available sources from step 1, it SHOULD attempt to retrieve
   the IANA root zone from another source on that list.  If the
   resolver has exhausted the list of sources, it SHOULD stop
   attempting to download the IANA root zone and SHOULD fall back to
   using regular DNS mechanisms for performing DNS resolutions.  Upon
   a failure, but before exhausting the list of available IANA root
   zone sources, the resolver MAY choose to cease attempting download
   the IANA root zone and if so it SHOULD fall back to using regular DNS
   mechanisms for performing DNS resolutions.

4. Having successfully downloaded a copy of the IANA root zone, the
   resolver MUST verify the contents of the IANA root zone using the
   ZONEMD {{RFC8976}} record contained within it.  Note that this
   REQUIRES verification of the ZONEMD record using DNSSEC {{BCP237}}
   and the configured IANA root zone trust anchor.  The contents of
   the fetched zone MUST NOT be used until after ZONEMD verification
   is complete and successful.  Once the zone data has been verified
   as the IANA root zone, the resolver can begin LocalRoot enabled DNS
   resolution, potentially using the steps
   defined in {{integrating-root-zone-data}}.  

5. The resolver MUST check the sources in step 1 at a regular interval
   to identify when a new copy of the IANA root zone is available.
   This internal MAY be configurable and SHOULD default to the IANA
   root zone's current SOA refresh value. When a resolver has detected
   that a new copy of the IANA root zone is available, the resolver
   should consider its copy stale and MUST start at step 1 to obtain a
   new zone.  Resolvers MAY check multiple sources to ensure one
   source has not fallen significantly behind in its copy of the IANA
   root zone.  Resolvers MUST have an upper limit beyond which if a
   new copy is not available it will revert to using regular DNS
   queries to the IANA root zone instead of continuing to use the
   previously downloaded copy.  This upper limit stale value MAY be
   configurable and SHOULD default to the root zone's current SOA
   expiry value.  Once the LocalRoot implementation's copy of the IANA
   root zone is no longer considered stale after a fresh copy has been
   obtained, the resolver may resume LocalRoot enabled resolution
   operations.

## Integrating root zone data into the resolution process

{: #integrating-root-zone-data }

# Operational Considerations

/* ED (WH): I don't think we can get away without describing how/where to pull
this information from at some point.  The ICANN https servers are one source,
or should resolver code bases use their own defined CDNs?

(WK): 100% agree. I personally think that this should be hosted on multiple
CDNs, and that expecting a single server or service to always be available
would be a massive mistake. But, I also don't think that resolvers should pull
from their own CDNs
- I don't want Acme Anvil and Resolvers (or their CDN!)  go out of business,
and have Acme Resolvers fail. This is (I believe) a sufficiently small amount
of data that hosting it on multiple CDNs should be trivial.... but, I also
believe that this topic should be discussed with the WG. */

/* Ed (WK): We might want to add some more discussions around failure handling,
but, 1:  {{RFC8806}} already covers much of this and 2: "don't teach your
grandmother to suck eggs" - implementations already handle this, so let's not
try to overspecify or over-constrain what they do. */

# Security Considerations

There are areas of potential concern that are mitigated to some extent
by using this mechanism.

The issue of leakage of potentially sensitive information
that may be contained in the query name used in DNS queries. Most root
servers (except b.root-servers.net) do not currently support queries
over encrypted transports, resulting in query names that are visible
to on-the-wire eavesdroppers, and may also be held in any operational
logs maintained by root server operators. Such concerns may be
mitigated by Query Name Minimization {{RFC9156}}, but common
implementations of this mechanism appear to only minimize query names
of four or fewer labels, and the uptake rate of query name
minimization appears to be quite low {{QNAMEMIN}}. Furthermore, even
with Query Name Minimization, queries for non-existent names
(generated from keyword searches and mis-configurations) can cause
additional privacy leaks.  {{RFC8806}} eliminates the need for the
resolver to perform specific queries to any root nameserver, and
obviates any such consideration of query name leakage
{{LOCALROOTPRIVACY}}.

The final issue solved with LocalRoot is that when information is
always available locally, usage of it is no longer subject to DDoS
attacks against the remote servers.  By having the answers effectively
permanently in cache, no queries to the upstream service provider
(such as root servers) are needed since {{RFC8806}} resolvers
effectively always have a cached set of data that is considered fresh
longer than the typical TTL records within the zone {{CACHEME}}
{{LOCALROOTPRIVACY}}.


# IANA Considerations

This document has no IANA actions.


--- back


# Acknowledgments
{:numbered="false"}

The authors have discussed this idea with many people, and have likely
forgotten to acknowledge and credit many of them. If we discussed this with
you, and you are not listed, please please let us know and we'll add you.

The authors would like to thank Joe Abley, Vint Cerf, John Crain, Marco Davids,
Paul Hoffman, Peter Koch, Matt Larson, Florian Obser, Swapneel Patnekar, Puneet
Sood, Robert Story, Ondrej Sury, Suzanne Woolf, and many many others for their
comments, suggestions and input.

In addition, one of the authors would like to once again thank the
bands "Infected Mushroom", "Kraftwerk", and "deadmau5" for providing
the soundtrack to which this was written.  Another author recently
discovered the band "Trampled by Turtles" and has submitted it as a
nomination for the best-band-name-ever award.

# Appendix A: Example Configurations
{:numbered="false"}

These examples are provided to show how the LocalRoot mechanism can be
configured in various resolver implementations. They are not intended to be
exhaustive, and may not work with all versions of the software.

/* Ed (WK): These examples are just to get started. We would appreciate
contributions from the resolver operators.

Yes, we are fully aware of the circular dependency of trying to resolve e.g
www.internic.net when bootstrapping. More discussion on serving the IANA root zone
over HTTP by IP will be added later. */

## ISC BIND 9.14 and above
{:numbered="false"}

See the BIND documentation for [mirror
zones](https://bind9.readthedocs.io/en/stable/reference.html#namedconf-statement-type%20mirror).


Example configuration using a "mirror" zone:

~~~
zone "." {
    type mirror;
};
~~~

## Knot Resolver
{:numbered="false"}

See the Knot Resolver [Cache
prefilling](https://knot-resolver.readthedocs.io/en/v5.0.1/modules-prefill.html?highlight=cache%20prefilling)
documentation for more information.

The following example configuration will prefill the IANA root zone using HTTPS:

~~~
modules.load('prefill')
prefill.config({
      ['.'] = {
              url = 'https://www.internic.net/domain/root.zone',
              interval = 86400  -- seconds
              ca_file = '/etc/pki/tls/certs/ca-bundle.crt', -- optional
      }
})
~~~

## Unbound 1.9 and above
{:numbered="false"}

See the Unbound documentation for [Authority Zone Options](https://unbound.docs.nlnetlabs.nl/en/latest/manpages/unbound.conf.html#unbound-conf-auth-url) configuration.

The following example configuration will prefill the IANA root zone using HTTPS:

~~~
auth-zone:
    name: "."
    url: "https://www.internic.net/domain/root.zone"
    zonefile: "root.zone"
    fallback-enabled: yes
    for-downstream: no
    for-upstream: yes
    zonefile: "root.zone"
~~~
