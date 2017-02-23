---
title: Serving Stale Data to Improve DNS Resiliency
docname: draft-tale-dnsop-serve-stale-00
date: 2017-02

ipr: trust200902
area: Internet
wg: DNSOP Working Group
kw: Internet-Draft
cat: std

coding: us-ascii
pi:
  - toc
  - symrefs
  - sortrefs

author:
  -
    ins: D.C. Lawrence
    name: David C Lawrence
    org: Akamai Technologies
    street: 150 Broadway
    city: Cambridge
    code: MA 02142-1054
    country: USA
    email: tale@akamai.com

  -
    ins: W. Kumari
    name: Warren Kumari
    org: Google
    street: 1600 Amphitheatre Parkway
    city: Mountain View
    code: CA 94043
    country: USA
    email: warren@kumari.net

--- abstract

This draft defines a method for recursive resolvers to use stale DNS
data to avoid outages when authoritative nameservers cannot be reached
to refresh expired data.

--- middle

# Introduction

Traditionally the Time To Live (TTL) of a DNS resource record has been
understood to represent the maximum number of seconds that a record
can be used before it must be discarded, based on its description and
usage in {{!RFC1035}} and clarifications in {{!RFC2181}}.
Specifically, {{!RFC1035}} Section 3.2.1 says that it "specifies the
time interval that the resource record may be cached before the source
of the information should again be consulted".

Notably, the original DNS specification does not say that data past
its expiration cannot be used.  This document proposes a method for
how recursive resolvers should handle stale DNS data to balance the
competing needs of resiliency and freshness. It is predicated on the
observation that authoritative server unavailability can cause outages
even when the underlying data those servers would return is typically
unchanged.

Several major recursive resolver operations currently use stale data
for answers in some way, including Akamai, OpenDNS, Xerocole, and
Google (I think).

# Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in {{!RFC2119}}

For a comprehensive treatment of DNS terms, please see {{!RFC7719}}.

# Description

Three notable timers drive considerations for the use of stale data,
as follows:

* A client response timer, which is the maximum amount of time a
recursive resolver should allow between the receipt of a resolution
request and sending its response.

* A query resolution timer, which caps the total amount of time a
recursive resolver spends processing the query.

* A maximum stale timer, which caps the amount of time
that records will be kept past their expiration.

Recursive resolvers already have the second timer; the first and
third timers are new concepts for this mechanism.

When a request is received by the recursive resolver, it SHOULD start
the client response timer.  This timer is used to avoid client
timeouts.  It SHOULD be configurable, with a recommended value of 1.8
seconds.

The resolver then checks its cache for an unexpired answer. If it
finds none and the Recursion Desired flag is not set in the request,
it SHOULD immediately return the response without consulting the
expired record cache.

If iterative lookups will be done, it SHOULD start the query
resolution timer.  This timer bounds the work done by the resolver,
and is commonly 30 seconds.

If the answer has not been completely determined when the client
response timer has elapsed, the resolver SHOULD then check its cache
to see whether there is expired data that would satisfy the request.
If so, it sends a response and the TTL fields of the expired records
SHOULD be set to 1.

The maximum stale timer is used for cache management and is
independent of the query resolution process. This timer is
conceptually different from the maximum cache TTL that exists in many
resolvers, the latter being a clamp on the value of TTLs as
received from authoritative servers.  The maximum stale timer SHOULD
be configurable, and defines the length of time after a record expires
that it SHOULD be retained in the cache.  The suggested value is 7
days, which gives time to notice the problem and and for human
intervention for fixing it.

# Implementation Caveats

Answers from authoritative servers that have a DNS Response Code of
either 0 (NOERROR) or 3 (NXDOMAIN) MUST be considered to have
refreshed the data at the resolver.  In particular, this means that
this method is not meant to protect against operator error at the
authoritative server that turns a name that is intended to be valid
into one that is non-existent, because there is no way for a resolver
to know intent.

Resolution is given a chance to succeed before stale data is used to
adhere to the original intent of the design of the DNS.  This
mechanism is only intended to add robustness to failures, and not be a
standard operational occurrence as would happen if stale data were
used immediately and then a cache refresh attempted after the client
response has been sent.

It is important to continue the resolution attempt after the stale
response has been sent because some pathological resolutions can take
at least a dozen seconds succeed as they cope with down servers, bad
networks, and other problems.  Stopping the resolution attempt when
the response has been sent would mean that answers in these
pathological cases would never be refreshed.

Canonical Name (CNAME) records mingled in the expired cache with other
records at the same owner name can cause surprising results.  This was
observed with an initial implementation in BIND, where a hostname
changed from having a CNAME record to an IPv4 Address (A) record.
BIND does not evict CNAMEs in the cache when other types are received,
which in normal operations is not an issue.  However, after both
records expired and the authorities became unavailable, the fallback
to stale answers returned the older CNAME instead of the newer A.

(This might apply to other occluding types, so more thought should be
given to the overall issue.)

Keeping records around after their normal expiration will of course
cause caches to grow larger than if records were removed at their TTL.
Specific guidance on managing cache sizes is outside the scope of this
document.

# Security Considerations

The most obvious security issue is the increased likelihood of DNSSEC
validation failures when using stale data because signatures could be
returned outside their validity period.  

Additionally, bad actors have been known to use DNS caches as a kind
of perpetual cloud database, keeping records alive even after their
authorities have gone away.  This makes that easier.

# Privacy Considerations

This document does not add any practical new privacy issues.

# NAT Considerations

The method described here is not affected by the use of NAT devices.

# IANA Considerations

This document contains no actions for IANA.

# Acknowledgements

The authors wish to thank Matti Klock for initial review.

--- back
