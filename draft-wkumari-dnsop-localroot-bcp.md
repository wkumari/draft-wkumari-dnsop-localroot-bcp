---
title: "Populating resolvers with the root zone."
abbrev: "LocalRoot"
category: std

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

normative:
  BCP237:
  RFC1982:  # SOA math
  RFC4033:  # DNSSEC
  RFC8198:
  RFC8499:
  RFC8806:
  RFC8976:

informative:
  RFC5936:  # DNS Zone Transfer
  RFC7766:  # DNS Transport over TCP
  RFC7706:
  RFC9156:  # QNAME Minimisation
  RFC9110:  # HTTP Semantics and Methods

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
  draft-hardaker-dnsop-dns-xfr-scheme:
    title: The DNS XFR URI Schemes
    target: https://datatracker.ietf.org/doc/draft-hardaker-dnsop-dns-xfr-scheme/
  draft-hardaker-dnsop-root-zone-publication-list-guidelines:
    title: Guidelines for IANA DNS Root Zone Publication List Providers
    target: https://raw.githubusercontent.com/hardaker/draft-hardaker-dnsop-root-zone-publication-list-guidelines/refs/heads/main/draft-hardaker-dnsop-root-zone-publication-list-guidelines.md
  NOROOTS:
    title: On Eliminating Root Nameservers from the DNS
    target: https://www.icir.org/mallman/pubs/All19b/All19b.pdf
  DNEROOTNAMES:
    title: NoError vs NxDomain by-week
    target: https://rssac002.root-servers.org/rcode_0_v_3.html

--- abstract

DNS recursive resolver operators need to provide the best service
possible for their users, which includes providing an operationally
robust and privacy protecting service.  Challenges to these deployment
goals include difficulty of getting responses from the root servers
(such as during a network attack), longer-than-desired round-trip
times to the closest DNS root server, and privacy issues relating to
queries sent to the DNS root servers.  Resolvers can solve all of
these issues by simply serving an already cached a copy of the full
root zone.

This document shows how resolvers can fetch, cache and maintain a copy
of the root zone, how to detect if the contents becomes stale, and
procedures for handling error conditions.

This document obsoletes {{RFC8806}}.

--- middle

# Introduction

DNS recursive resolvers have to provide answers to all queries from
their clients, even those for domain names that do not exist.  For
each queried name that is within a top-level domain (TLD) that is not
in the recursive resolver's cache, the resolver must send a query to a
root server to get the information for that TLD or to find out that
the TLD does not exist.  Many of the queries from recursive resolvers
to root servers get answers that are referrals to other servers.  But,
research shows that the vast majority of queries going to the root are
for names that do not exist in the root zone {{DNEROOTNAMES}}.  There
are privacy implications for queries that can be observed by malicious
third parties for both the queries for names that do exist and for the
names that do not exist.

## Local Caching of Root Server Data {#goals}

Caching the root zone data locally, commonly referred to as running a
"LocalRoot" instance, provides a method for the operator of a
recursive resolver to use a complete copy of the IANA root zone
locally instead of sending requests to the Root Server System (RSS).
This technique can be implemented using a number of different
implementation techniques, including as described in this document,
but the net effect is the same: few, if any, queries should be sent to
the actual RSS.

Two potential implementation mechanisms are documented herein for
achieving LocalRoot functionality: by having the resolver pre-fetch
the root zone at regular intervals and populate its cache with
information, or by running an authoritative server in parallel with
the recursive resolver that acts as a local authoritative root server.
Other mechanisms for implementing LocalRoot functionality MAY be used.
To a client, the net effect of using any technique SHOULD be nearly
indistinguishable to that of a non-Localroot resolver.

Note that enabling LocalRoot functionality in a resolver should have
little effect on improving resolver speed to its stub resolver clients
for queries under Top Level Domains (TLDs), as the TTL for most TLDs
is long-lived (two days in the current root zone). Thus, most TLD
nameserver and address records are typically already in a resolver's
cache.  Negative answers from the root servers are also cached in a
similar fashion, though potentially for a shorter time based on the
SOA negative cache timing (one day in the current root zone).

Also note that a different approach to partially mitigating some of
the privacy problems that a LocalRoot enabled resolver solves can be
achieved using the "Aggressive Use of DNSSEC-Validated Cache"
{{RFC8198}} functionality.

Readers are expected to be familiar with the terminology defined in
{{RFC8499}}.

This behavior SHOULD be used by all general-purpose recursive
resolvers used on the public Internet.

# Conventions and Definitions {#definitions}

{::boilerplate bcp14-tagged}

## Terminology used in this document

* IANA root zone: the DNS root zone as published by IANA.  {{RFC8499}}
  describes the same root zone as "The zone of a DNS-based tree whose
  apex is the zero- length label.  Also sometimes called ''the DNS
  root'."

* A LocalRoot enabled resolver: a recursive resolver that makes use of
  a local copy of the root zone data by any means possible.

* A LocalRoot implementation: the software or system of software
  responsible for implementing the functionality described in this
  specification.  Note that this may be done in a single piece of
  software, or a set of components, or even a set of systems as long
  as the functionality of a LocalRoot implementation behaves
  identically to a resolver that is not Localroot enabled.

# Making LocalRoot functionality the recommended practice

(Ed: this section is primarily here to drive discussion within IETF
WGs of interest and will be removed (or its text moved) prior to
publication, with the possible exception of the list of differences
between this document and RFC8806)

Note: DNSOP needs to discuss whether to publish this as a BCP or as a
proposed standard.  Currently this is listed as STD track based on a
number of preliminary conversations the authors had with both
operators and IETF participants.

{{RFC8806}} is an Informational document that describes a mechanism
that resolver operators can use to improve the performance,
reliability, and privacy of their resolvers.  This document concludes
the concept of {{RFC8806}} was a success, but that actual
implementation of it has varied according to the needs of various code
bases and operational environments.

Secure DNS verification of an obtained copy of the IANA root zone is
possible because of the use of the RSS's ZONEMD {{RFC8976}} record.
This allows for the entire zone to be fetched and subsequently
verified before being used within recursive resolvers resolvers.
DNSSEC provides the same assurance for individual signed resource
records sourced from the root zone, including of the ZONEMD record
itself.

This document:

1. promotes the behavior in {{RFC8806}} to be either a Proposed
   standard or a Best Current Practice, depending on what the WG
   decides.
2. RECOMMENDS that resolver implementations provide a simple configuration
   option to enable or disable functionality, and
3. RECOMMENDS that resolver implementations enable this behavior by default, and
4. REQUIRES that {{RFC8976}} be used to validate the IANA root zone information
   before loading it.
5. Adds a mechanism for priming the list of places for fetching root zone data.
6. Adds protocol steps for ensuring resolution stability and resiliency.

(ed note: should we require checking the SOA serial number for
increasing values in #4, and down below in localroot enabled resolver
requirements?)

# LocalRoot enabled resolver requirements {#requirements}

In order to implement the functionality described in this document:

- A LocalRoot implementation MUST have a configured DNSSEC trust
  anchor as an up-to-date copy of the public part of the Key Signing
  Key (KSK) {{RFC4033}} or used to sign the DNS root or its DS record.

- A LocalRoot implementation MUST validate the contents of the root zone using
  ZONEMD {{RFC8976}}, and MUST check the validity of the ZONEMD record
  using DNSSEC.

- A LocalRoot implementation MUST retrieve or be provisioned with a
  copy of the entire current root zone (including all DNSSEC-related
  records) (see {{protocol-steps}}).

- A LocalRoot implementation MUST be able to fall back to querying the
  authoritative RSS servers whenever the local copy of the root zone
  data is unavailable or has been deemed stale (see {{protocol-steps}}).

- A LocalRoot implementation MUST return records from the root zone
  without modification.

- A LocalRoot enabled resolver return identical answer content about
  the DNS root (or any other part of the DNS) as if it would if it
  were not operating as a LocalRoot enabled resolver.

## Important change from RFC8806

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
{{RFC8806}}.  This can include the implementation guidance described
in RFC8806, but this document allows for implementations to select any
mechanism for fetching and re-distributing the contents of the root
zone on their resolver service addresses as long as the other
requirements specified in this document are still followed (see
{{requirements}}).

For example, an implementation can simply "prefill" the resolver's
cache with the current contents of the root zone. As the resulting
behavior is (essentially) indistinguishable from the mechanism defined
in RFC8806, this is viewed as being an acceptable implementation
decision.

# Functionality of a LocalRoot enabled resolver

To implement the goals described in {{goals}} and meet the
requirements described in {{requirements}}, a LocalRoot enabled
resolver will need to perform three fundamental tasks:

1. Identify locations from where root zone data can be obtained
   {{root-zone-sources}}.
2. Downloading and refreshing the root zone data from one of the
   publication points {{protocol-steps}}.
3. Integrating and serving the data while performing DNS resolutions
   {{integrating-root-zone-data}}

Each of these are described in the subsections below.  Note that
implementations may vary significantly in how these tasks are
performed, ranging from static configuration to more active systems.

## Identifying locations from where root zone data can be obtained {#root-zone-sources}

For a LocalRoot enabled resolver to serve up to date data, an
implementation must be able to fetch the contents of the entire IANA
root zone on a regular basis from at least one publication source.
Implementations can find sources of root zone data in a number of
ways, including but not limited to:

1. An operationally configured list of sources (for example a
   file of URLs) that can be used to fetch a copy of the IANA root
   zone.
2. A list of sources distributed with the resolver software itself,
   (akin to how the root hints file is distributed with many resolvers
   today).
3. Downloading a list of available sources from the IANA using the
   sources described in {{iana-root-zone-list}}.

To support LocalRoot implementations, IANA will aggregate, publish and
maintain a list of IANA DNS root zone sources at *TBD-URL*
{{iana-root-zone-list}}.  Guidance to IANA (or for other entities
wishing to collect and redistribute a list of sources) for how to
collect and maintain a list of IANA root data publication sources is
discussed separately in
{{draft-hardaker-dnsop-root-zone-publication-list-guidelines}}.


## Downloading and refreshing root zone data {#protocol-steps}

When retrieving a copy of the IANA root zone from a list of IANA root
zone publication points, a LocalRoot enabled implementation MAY be use
the following steps.  Note that as long as the desired effect of
performing normal DNS resolution remains stable when combined with
LocalRoot functionality, other implementations MAY be used.

If a local copy of the IANA root zone data is unavailable for use in
DNS resolution at any point in these steps, resolvers SHOULD fall back
to performing DNS resolution by issuing queries directly to the RSS
instead.  If a resolver is unable to do so, it MUST respond to client
requests with a SERVFAIL response code.

1. The resolver SHOULD use a list of root zone sources identified in
   {{root-zone-sources}} for obtaining a copy of the IANA root zone.

2. The resolver MUST select one of the available sources from step 1,
   and from it retrieve a current copy of the IANA root zone.
   Resolvers SHOULD prioritize sources they can fetch the most
   efficiently.  For example, when supported, https sources should be
   preferred as it allows for compression negotiation as well as the
   possibility of using low-cost, well-distributed CDNs to distribute
   the zone files.  When querying a source of IANA root zone data, the
   resolver SHOULD minimize impact to the source by querying at a rate
   no faster than specified by the SOA refresh timer and SHOULD use
   data freshness protocol checks over downloading the entire contents
   at each refresh (example checks include the HEAD method {{RFC9110}}
   when using HTTP(s) or by querying the root zone's SOA over DNS
   first when using AXFR, IXFR or XoT).  Once fetched, an
   implementation MUST NOT make use of an obtained IANA root zone with
   a SOA serial number less than any previously obtained copy
   {{RFC1982}}.

3. If the resolver failed to retrieve the IANA root zone content in
   step 2, or the zone content's serial number was deemed to be older
   than an already cached copy then it SHOULD attempt to retrieve the
   IANA root zone from another source on that list if there are other
   available sources from available sources from step 1.  If the
   resolver has exhausted the list of sources, it SHOULD stop
   attempting to download the IANA root zone and wait another refresh
   time length until retrying all of the sources again.

4. Having successfully downloaded a copy of the IANA root zone, the
   resolver MUST verify the contents of the IANA root zone using the
   ZONEMD {{RFC8976}} record contained within it.  Note that this
   REQUIRES verification of the ZONEMD record using DNSSEC {{BCP237}}
   and the configured IANA root zone trust anchor.  The contents of
   the fetched zone MUST NOT be used until after ZONEMD verification
   is complete and successful.  Once the zone data has been verified
   as the IANA root zone, the resolver can begin LocalRoot enabled DNS
   resolution, potentially using the steps defined in
   {{integrating-root-zone-data}}.

5. The resolver MUST check the sources in step 1 at a regular interval
   to identify when a new copy of the IANA root zone is available.
   This internal MAY be configurable and SHOULD default to the IANA
   root zone's current SOA refresh value. When a resolver has detected
   that a new copy of the IANA root zone is available, the resolver
   SHOULD start at step 1 to obtain a new zone.  Resolvers MAY check
   multiple sources to ensure one source has not fallen significantly
   behind in its copy of the IANA root zone.

Resolvers MUST have an upper limit beyond which if a new copy is not
available it will revert to using regular DNS queries to the IANA root
zone instead of continuing to use the previously downloaded copy.  If
at any point this upper limit has been reached, the resolver SHOULD
fall back to using regular DNS mechanisms for performing DNS
resolutions on behalf of its clients.  This upper limit value MAY be
configurable and SHOULD default to the root zone's current SOA expiry
value.  Once the LocalRoot implementation's copy of the IANA root zone
has been successfully refreshed and is no longer considered expired,
the resolver may resume LocalRoot enabled resolution operations.

## Integrating and serving root zone data during resolution

{: #integrating-root-zone-data }

Any mechanism that a recursive resolver can use to serve the data
obtained in {{protocol-steps}} in such a way that it is functionally
indishinguishable to a client from having followed regular DNS
resolution processes should be considered an acceptable
implementation.  Two example implementation descriptions are included
in the following two subsections.

### Pre-caching the root zone data

Once the root zone data has been collected and verified as complete
and correct ({{protocol-steps}}), a resolver MAY simply update its
cache with the newly obtained values.  This functionally entirely
alleviates the need for sending any (other) DNS requests to the RSS.

### Running a local authoratative copy of the root zone in parallel

{{RFC8806}} described an implementation mechanism where a copy of the
root zone could be run in an authoratative server running in parallel
to the recursive resolver.  The recursive resolver could then be
configured to simply point at this parallel server for obtaining data
related to the root zone instead of the RSS itself.  Note that
{{RFC8806}} required that the parallel server be running on a loopback
address and this specification removes that requirement and allows the
parallel service to run on any address it can legitimately be used on.
Note that the server MUST NOT use an address of one of the official
root server addresses in the root zone.

# IANA maintained list of root zone publication points  {#iana-root-zone-list}

WJH:TODO

This list of IANA root zone data publication points available at
TBD-URL may be used when downloading and refreshing the root zone
data, as described in {{protocol-steps}}.  Specifically, this IANA DNS
root zone publication list MAY be used by the resolver software
directly, or by the operating system a resolver is deployed on, or by
a network operator when configuring a resolver.

The contents of the IANA DNS root publication points file MUST
verified as to its integrity as having come from IANA and MUST be
verified as complete.

## root zone publication points

NOTE: this is but an example format that is expected to spur
discussions within IETF working groups like DNSOP.  Whether this is a
list in a simple line-delimited format like below or signed JSON or
signed PGP or ... is subject to debate.

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

## Publication point operational considerations

Implementations SHOULD optimize retrieval to minimize impacts on the
server.  Because the list is not expected to change frequently,
implementations SHOULD refrain from querying the IANA source more than
once a week.

# Operational Considerations

TBD

# Security Considerations

There are areas of potential concern that are mitigated to some extent
by using this mechanism.

## Leakage of potentially sensitive information

One privacy concern with the use of DNS is the leakage of potentially
sensitive information that may be contained in the query name used in
DNS queries. Most root servers (except b.root-servers.net) do not
currently support queries over encrypted transports, resulting in
query names that are visible to on-the-wire eavesdroppers, and may
also be held in any operational logs maintained by root server
operators. Such concerns may be mitigated by Query Name Minimization
{{RFC9156}}, but common implementations of this mechanism appear to
only minimize query names of four or fewer labels, and the uptake rate
of query name minimization appears to be quite low
{{QNAMEMIN}}. Furthermore, even with Query Name Minimization, queries
for non-existent names (generated from keyword searches and
mis-configurations) can cause additional privacy leaks.  {{RFC8806}}
eliminates the need for the resolver to perform specific queries to
any root nameserver, and obviates any such consideration of query name
leakage {{LOCALROOTPRIVACY}}.

## Local resiliency of the DNS

Another issue solved with LocalRoot is that when information is always
available locally, usage of it is no longer subject to DDoS attacks
against the remote servers.  By having the answers effectively
permanently in cache, no queries to the upstream service provider
(such as root servers) are needed since {{RFC8806}} resolvers
effectively always have a cached set of data that is considered fresh
longer than the typical TTL records within the zone {{CACHEME}}
{{LOCALROOTPRIVACY}}.


# IANA Considerations

TBD: describe the request for IANA to support a list of root server
publication points at TBD-URL.

--- back


# Acknowledgments
{:numbered="false"}

The authors have discussed this idea with many people, and have likely
forgotten to acknowledge and credit many of them. If we discussed this with
you, and you are not listed, please please let us know and we'll add you.

This work has been founded upon previous documents.  These include
{{RFC7706}} and {{RFC8806}} authored by Warren Kumari and Paul Hoffman
and "On Eliminating Root Nameservers from the DNS" {{NOROOTS}} by Mark
Allman.

The authors would like to thank Joe Abley, Vint Cerf, John Crain,
Marco Davids, Peter Koch, Matt Larson, Florian Obser, Swapneel
Patnekar, Puneet Sood, Robert Story, Ondrej Sury, Suzanne Woolf, and
many many others for their comments, suggestions and input to both
past and current versions of this document.

In addition, one of the authors would like to once again thank the
bands "Infected Mushroom", "Kraftwerk", and "deadmau5" for providing
the soundtrack to which this was written.  Another author recently
discovered the band "Trampled by Turtles" while working on this
document and is submitting it as a nomination for the
best-band-name-ever award.
