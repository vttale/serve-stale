---
title: Serving Stale Data to Improve DNS Resiliency
abbrev: DNS Serve Stale
docname: draft-ietf-dnsop-serve-stale-02
date: 2018-10

ipr: trust200902
area: Internet
wg: DNSOP Working Group
kw: Internet-Draft
cat: std
updates: 1034, 1035

coding: us-ascii
pi:
  - toc
  - symrefs
  - sortrefs

author:
  -
    ins: D.C. Lawrence
    name: David C Lawrence
    org: Oracle + Dyn
    email: tale@dd.org

  -
    ins: W. Kumari
    name: Warren "Ace" Kumari
    org: Google
    street: 1600 Amphitheatre Parkway
    city: Mountain View
    code: CA 94043
    country: USA
    email: warren@kumari.net

  -
    ins: P. Sood
    name: Puneet Sood
    org: Google
    email: puneets@google.com

informative:
  DikeBreaks:
    target: https://www.isi.edu/~johnh/PAPERS/Moura18b.pdf
    title: "When the Dike Breaks: Dissecting DNS Defenses During DDos"
    author:
      -
        name: Giovane C. M. Moura
      -
        name: John Heidemann
      -
        name: Moritz Mueller
      -
        name: Ricardo de O. Schmidt
      -
        name: Marco Davids
    date: 2018-10-31
    seriesinfo:
      ACM: 2018 Internet Measurement Conference
      DOI: 10.1145/3278532.3278534

--- abstract

This draft defines a method for recursive resolvers to use stale DNS
data to avoid outages when authoritative nameservers cannot be reached
to refresh expired data.  It updates the definition of TTL from
{{!RFC1034}} and {{!RFC1035}} to make it clear that data can be kept
in the cache beyond the TTL expiry and used for responses when a
refreshed answer is not readily available.

--- note_Ed_note

Text inside square brackets (\[\]) is additional background
information, answers to frequently asked questions, general musings,
etc.  They will be removed before publication.  This document is being
collaborated on in GitHub at
\<https://github.com/vttale/serve-stale\>.  The most recent
version of the document, open issues, etc should all be available
here.  The authors gratefully accept pull requests.

--- middle

# Introduction

Traditionally the Time To Live (TTL) of a DNS resource record has been
understood to represent the maximum number of seconds that a record
can be used before it must be discarded, based on its description and
usage in {{!RFC1035}} and clarifications in {{!RFC2181}}.

This document proposes that the definition of the TTL be explicitly
expanded to allow for expired data to be used in the exceptional
circumstance that a recursive resolver is unable to refresh the
information.  It is predicated on the observation that authoritative
server unavailability can cause outages even when the underlying data
those servers would return is typically unchanged.

We describe a method below for this use of stale data, balancing the
competing needs of resiliency and freshness. 

# Notes to readers

\[ RFC Editor, please remove this section before publication!  Readers:
This is conversational text to describe what we've done, and will be
removed, please don't bother sending editorial nits. :-) \]

Due to circumstances, the authors of this document got sidetracked,
and we lost focus.  We are now reviving it, and are trying to address
and incorporate comments.  There has also been more deployment and
implementation recently, and so this document is now more describing
what is known to work instead of simply proposing a concept.

# Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in
BCP 14 {{!RFC2119}} {{!RFC8174}} when, and only when, they appear in all
capitals, as shown here.

For a comprehensive treatment of DNS terms, please see {{?RFC7719}}.

# Background

There are a number of reasons why an authoritative server may become
unreachable, including Denial of Service (DoS) attacks, network
issues, and so on.  If the recursive server is unable to contact the
authoritative servers for a query but still has relevant data that has
aged past its TTL, that information can still be useful for generating
an answer under the metaphorical assumption that "stale bread is
better than no bread."

{{!RFC1035}} Section 3.2.1 says that the TTL "specifies the time
interval that the resource record may be cached before the source of
the information should again be consulted", and Section 4.1.3 further
says the TTL, "specifies the time interval (in seconds) that the
resource record may be cached before it should be discarded."

A natural English interpretation of these remarks would seem to be
clear enough that records past their TTL expiration must not be used.
However, {{RFC1035}} predates the more rigorous terminology of
{{RFC2119}} which softened the interpretation of "may" and "should".

{{RFC2181}} aimed to provide "the precise definition of the Time to
Live", but in Section 8 was mostly concerned with the numeric range of
values and the possibility that very large values should be capped. (It
also has the curious suggestion that a value in the range 2147483648
to 4294967295 should be treated as zero.)  It closes that section by
noting, "The TTL specifies a maximum time to live, not a mandatory
time to live."  This is again not {{RFC2119}}-normative language, but
does convey the natural language connotation that data becomes
unusable past TTL expiry.

Several major recursive resolver operators currently use stale data
for answers in some way, including Akamai (via both Nomimum and
Xerocole), BIND, Knot, OpenDNS, and Unbound.  Apple can also use stale
data as part of the Happy Eyeballs algorithms in mDNSResponder.  The
collective operational experience is that it provides significant
benefit with minimal downside.

# Standards Action

The definition of TTL in {{RFC1035}} Sections 3.2.1 and 4.1.3 is
amended to read:

TTL
: a 32 bit unsigned integer number of seconds in the range 0 -
2147483647 that specifies the time interval that the resource record
MAY be cached before the source of the information MUST again be
consulted.  Zero values are interpreted to mean that the RR can only
be used for the transaction in progress, and should not be cached.
Values with the high order bit set SHOULD be capped at no more than
2147483647.  If the authority for the data is unavailable when
attempting to refresh the data past the given interval, the record MAY
be used as though it is unexpired.

\[ Discussion point: capping values with the high order bit as being
max positive, rather than 0, is a change from {{RFC2181}}.  Also, we
could use this opportunity to recommend a much more sane maximum value
like 604800 seconds, one week, instead of the literal maximum of 68
years. \]



# Example Method

There is conceivably more than one way a recursive resolver could
responsibly implement this resiliency feature while still respecting
the intent of the TTL as a signal for when data is to be refreshed.

In this example method three notable timers drive considerations for
the use of stale data, as follows:

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
cache for expired records.

If iterative lookups will be done, it SHOULD start the query
resolution timer.  This timer bounds the work done by the resolver,
and is commonly around 10 to 30 seconds.

If the answer has not been completely determined by the time the
client response timer has elapsed, the resolver SHOULD then check its
cache to see whether there is expired data that would satisfy the
request.  If so, it adds that data to the response message and SHOULD
set the TTL of each expired record in the message to 30 seconds.  The
response is then sent to the client while the resolver continues its
attempt to refresh the data.  30 second was chosen because
historically 0 second TTLs have been problematic for some
implementations, and similarly very short TTLs could lead to
congestive collapse as TTL-respecting clients rapidly try to refresh.
30 seconds not only sidesteps those potential problems with no
practical negative consequence, it would also rate limit further
queries from any client that is honoring the TTL, such as a forwarding
resolver. 

The maximum stale timer is used for cache management and is
independent of the query resolution process. This timer is
conceptually different from the maximum cache TTL that exists in many
resolvers, the latter being a clamp on the value of TTLs as received
from authoritative servers.  The maximum stale timer SHOULD be
configurable, and defines the length of time after a record expires
that it SHOULD be retained in the cache.  The suggested value is 7
days, which gives time to notice the resolution problem and for human
intervention to fix it.

This same basic technique MAY be used to handle stale data associated
with delegations.  If authoritative server addresses are not able to
be refreshed, resolution can possibly still be successful if the
authoritative servers themselves are still up.

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
mechanism is only intended to add robustness to failures, and to be
enabled all the time.  If stale data were used immediately and then a
cache refresh attempted after the client response has been sent, the
resolver would frequently be sending data that it would have had no
trouble refreshing.  As modern resolvers use techniques like
pre-fetching and request coalescing for efficiency, it is not
necessary that every client request needs to trigger a new lookup flow
in the presence of stale data, but rather than a good-faith effort
have been recently made to refresh the stale data before it is
delivered to any client.  The recommended period between attempting
refreshes is 30 seconds.

It is important to continue the resolution attempt after the stale
response has been sent, until the query resolution timeout, because
some pathological resolutions can take many seconds to succeed as they
cope with unavailable servers, bad networks, and other problems.
Stopping the resolution attempt when the response with expired data
has been sent would mean that answers in these pathological cases
would never be refreshed.

Canonical Name (CNAME) records mingled in the expired cache with other
records at the same owner name can cause surprising results.  This was
observed with an initial implementation in BIND when a hostname
changed from having an IPv4 Address (A) record to a CNAME.  The
version of BIND being used did not evict other types in the cache when
a CNAME was received, which in normal operations is not a significant
issue.  However, after both records expired and the authorities became
unavailable, the fallback to stale answers returned the older A
instead of the newer CNAME.

Keeping records around after their normal expiration will of course
cause caches to grow larger than if records were removed at their TTL.
Specific guidance on managing cache sizes is outside the scope of this
document.  

## Implementation considerations

This document mainly describes the method / concept behind serving stale data, and intentionally does not provide a formal algorithm. The concept is not overly complex, and we believe that it would be hubris to believe that we know better than resolver authors how exactly to implement the concept in their code-base. The processing of serve-stale is a local operation, and consitent variables between implementations (or instances) is not needed for interoperability. 

However, we would like to highlight the impact of various knobs / variables [Ed: .. and the WG asked for this :-)]

The most obvious of these is the "maximum stale timer". If this variable is too large, it could cause excessive cache memory utilization. This could be mitigated by prioritizing removal of these record over non-expired records during cache exhaustion. Implmentations may wish to consider whether to track the popularity of names in client requests versus simply tracking age, and whether to provide a feature for manually flushing only stale records. If this variable is too small, the serve-stale technique becomes less effective, as the record may not be in the cache to be used if needed. One nice property is that more popular names are, by definition, queried more often than unpopular names - this means that resetting a timer each time a name is used can automatically prioritize popular names over unpopular ones. 

The "client response timer" is another variable which deserves consideration. If this value is too short, there exists the risk that stale answers may be used even when the authoritative server is actually reachable but slow; this may result in suboptimal answers being returned. Conversely, waiting too long will negatively impact the user response. 

Many resolvers implement prefetching of answers before the TTL has expired. If this has been attempted recently, and the authorative servers were determined to be not reachable at this time, the check can be skipped - the authors recommend that this value be the same as the check interval (see below). 

Another variable is how often to re-check if the authoritative server is available once it has initially been determined to be unreachable. If this variable is set too large, stale answers may continue to be returned even after the authoritative server is reachable. One of the motivations for serve-stale is to make the DNS more resilient to DOS attacks (and thereby make them less attractive as an attack vector). If the serve-stale technique becomes widely deployed, and this variable is too small, authoritative servers may be hit with a significant amount of traffic when they become reachable again. For this reason, the authors strongly recommend that this value not be set lower than 30 seconds. 

# Implementation Status

\[RFC Editor: per RFC 6982 this section should be removed prior to
publication.\]

The algorithm described in the {{example-method}} section was
originally implemented as a patch to BIND 9.7.0.  It has been in
production on Akamai's production network since 2011, and effectively
smoothed over transient failures and longer outages that would have
resulted in major incidents.  The patch was contributed to Internet
Systems Consortium and the functionality is now available in BIND 9.12
via the options stale-answer-enable, stale-answer-ttl, and max-stale-ttl.

Unbound has a similar feature for serving stale answers, but it works
in a very different way by returning whatever cached answer it has
before trying to refresh expired records. This is unfortunately not
faithful to the ideal that data past expiry should attempt to be
refreshed before being served.

Knot Resolver has an demo module here:
https://knot-resolver.readthedocs.io/en/stable/modules.html#serve-stale

Details of Apple's implementation are not currently known.

In the research paper "When the Dike Breaks: Dissecting DNS Defenses
During DDoS" {{DikeBreaks}}, the authors detected some use of stale answers by
resolvers when authorities came under attack.  Their research results
suggest that more widespread adoption of the technique would
significantly improve resiliency for the large number of requests that
fail or experience abnormally long resolution times during an attack.

# Security Considerations

The most obvious security issue is the increased likelihood of DNSSEC
validation failures when using stale data because signatures could be
returned outside their validity period.  This would only be an issue
if the authoritative servers are unreachable, the only time the
techniques in this document are used, and thus does not introduce
a new failure in place of what would have otherwise been success.

Additionally, bad actors have been known to use DNS caches to keep
records alive even after their authorities have gone away.  This
potentially makes that easier, although without introducing a new
risk.

# Privacy Considerations

This document does not add any practical new privacy issues.

# NAT Considerations

The method described here is not affected by the use of NAT devices.

# IANA Considerations

IANA is requested to assign an EDNS Option Code (as described in
Section 9 of [RFC6891]) for the serve-stale option specified in this
document.

# Acknowledgements

The authors wish to thank Robert Edmonds, Tony Finch, Bob Harold, Matti Klock, Jason Moreau, Giovane Moura,  Jean Roy, Mukund Sivaraman, Davey Song, Paul Vixie, Ralf Weber and Paul Wouters for their review and feedback.

--- back
