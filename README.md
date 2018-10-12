**Important:** Read CONTRIBUTING.md before submitting feedback or contributing
```




DNSOP Working Group                                          D. Lawrence
Internet-Draft                                       Akamai Technologies
Updates: 1034, 1035 (if approved)                              W. Kumari
Intended status: Standards Track                                 P. Sood
Expires: April 1, 2019                                            Google
                                                      September 28, 2018


              Serving Stale Data to Improve DNS Resiliency
                    draft-ietf-dnsop-serve-stale-01

Abstract

   This draft defines a method for recursive resolvers to use stale DNS
   data to avoid outages when authoritative nameservers cannot be
   reached to refresh expired data.

Ed note

   Text inside square brackets ([]) is additional background
   information, answers to frequently asked questions, general musings,
   etc.  They will be removed before publication.  This document is
   being collaborated on in GitHub at <https://github.com/vttale/serve-
   stale>.  The most recent version of the document, open issues, etc
   should all be available here.  The authors gratefully accept pull
   requests.

Status of This Memo

   This Internet-Draft is submitted in full conformance with the
   provisions of BCP 78 and BCP 79.

   Internet-Drafts are working documents of the Internet Engineering
   Task Force (IETF).  Note that other groups may also distribute
   working documents as Internet-Drafts.  The list of current Internet-
   Drafts is at https://datatracker.ietf.org/drafts/current/.

   Internet-Drafts are draft documents valid for a maximum of six months
   and may be updated, replaced, or obsoleted by other documents at any
   time.  It is inappropriate to use Internet-Drafts as reference
   material or to cite them other than as "work in progress."

   This Internet-Draft will expire on April 1, 2019.

Copyright Notice

   Copyright (c) 2018 IETF Trust and the persons identified as the
   document authors.  All rights reserved.

   This document is subject to BCP 78 and the IETF Trust's Legal
   Provisions Relating to IETF Documents
   (https://trustee.ietf.org/license-info) in effect on the date of
   publication of this document.  Please review these documents
   carefully, as they describe your rights and restrictions with respect
   to this document.  Code Components extracted from this document must
   include Simplified BSD License text as described in Section 4.e of
   the Trust Legal Provisions and are provided without warranty as
   described in the Simplified BSD License.

Table of Contents

   1.  Introduction
   2.  Notes to readers
   3.  Terminology
   4.  Background
   5.  Standards Action
   6.  EDNS Option
     6.1.  Option Format
   7.  Example Method
   8.  Implementation Caveats
   9.  Implementation Status
   10. new section
   11. Security Considerations
   12. Privacy Considerations
   13. NAT Considerations
   14. IANA Considerations
   15. Acknowledgements
   16. References
     16.1.  Normative References
     16.2.  Informative References
   Authors' Addresses

1.  Introduction

   Traditionally the Time To Live (TTL) of a DNS resource record has
   been understood to represent the maximum number of seconds that a
   record can be used before it must be discarded, based on its
   description and usage in [RFC1035] and clarifications in [RFC2181].

   This document proposes that the definition of the TTL be explicitly
   expanded to allow for expired data to be used in the exceptional
   circumstance that a recursive resolver is unable to refresh the
   information.  It is predicated on the observation that authoritative
   server unavailability causes outages even when the underlying data
   those servers would return is typically unchanged.

   A method is described for this use of stale data, balancing the
   competing needs of resiliency and freshness.  While this is intended
   to be immediately useful to the installed base of DNS software, an
   [RFC6891] EDNS option is also proposed in this document for enhanced
   signalling around the use of stale data by implementations that
   understand it.

   Note that it is highly desirable to couple this with a

2.  Notes to readers

   [ RFC Editor, please remove this section before publication!
   Readers: This is conversational text to describe what we've done, and
   will be removed, please don't bother sending editorial nits :-) ]

   Due to circumstances, the authors of this document got sidetracked,
   and we lost focus.  We are now reviving it, and are trying to address
   / incorporate comments.  There has also been more deployment and
   implementation recently, and so this document is now more describing
   what is known to work instead of simply proposing a concept. ]

   Open questions / notes:

   o  As we know that a number of implementations are doing something
      like serve-stale without any signalling, we are proposing removing
      the signalling, other than a "please do not do serve-stale EDNS"
      option for debugging.

   o  The TTL value to set in stale answers returned by recursive
      resolvers

   o

3.  Terminology

   The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
   "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
   "OPTIONAL" in this document are to be interpreted as described in
   [RFC2119] when, and only when, they appear in all capitals, as shown
   here.

   For a comprehensive treatment of DNS terms, please see [RFC7719].

4.  Background

   There are a number of reasons why an authoritative server may become
   unreachable, including Denial of Service (DoS) attacks, network
   issues, and so on.  If the recursive server is unable to contact the
   authoritative servers for a name but still has relevant data that has
   aged past its TTL, that information can still be useful for
   generating an answer under the metaphorical assumption that, "stale
   bread is better than no bread."

   [RFC1035] Section 3.2.1 says that the TTL "specifies the time
   interval that the resource record may be cached before the source of
   the information should again be consulted", and Section 4.1.3 further
   says the TTL, "specifies the time interval (in seconds) that the
   resource record may be cached before it should be discarded."

   A natural English interpretation of these remarks would seem to be
   clear enough that records past their TTL expiration must not be used,
   However, [RFC1035] predates the more rigorous terminology of
   [RFC2119] which softened the interpretation of "may" and "should".

   [RFC2181] aimed to provide "the precise definition of the Time to
   Live" but in Section 8 was mostly concerned with the numeric range of
   values and the possibility that very large values should be capped.
   (It also has the curious suggestion that a value in the range
   2147483648 to 4294967295 should be treated as zero.)  It closes that
   section by noting, "The TTL specifies a maximum time to live, not a
   mandatory time to live."  This is again not [RFC2119]-normative
   language, but does convey the natural language connotation that data
   becomes unusable past TTL expiry.

   Several major recursive resolver operations currently use stale data
   for answers in some way, including Akamai, OpenDNS, Xerocole, and
   Nominum.  Their collective operational experience is that it provides
   significant benefit with minimal downside.  As these implementations
   currently do this with no explicit signalling from the recursive to
   authoritative, nor from the client to the recursive, this document
   takes the position that explicit signalling is not required.
   [Editor: After discussions with implementors], and in light of the
   "DNS Camel" discussions, we are opting for implementation simplicity
   over more knobs, bells and whistles. ]

5.  Standards Action

   The definition of TTL in [RFC1035] Sections 3.2.1 and 4.1.3 is
   amended to read:

   TTL:  a 32 bit unsigned integer number of seconds in the range 0 -
      2147483647 that specifies the time interval that the resource
      record MAY be cached before the source of the information MUST
      again be consulted.  Zero values are interpreted to mean that the
      RR can only be used for the transaction in progress, and should
      not be cached.  Values with the high order bit set SHOULD be
      capped at no more than 2147483647.  If the authority for the data
      is unavailable when attempting to refresh the data past the given
      interval, the record MAY be used as though it has a remaining TTL
      of 1 second.

6.  EDNS Option

   The answer-of-last-resort can be achieved with changes only to
   resolvers, explicit signalling about the use of stale data can be
   done with an EDNS [RFC6891] option.  This EDNS option can be included
   from a stub to a recursive, explicitly signalling that it does NOT
   want stale answers.  It is expected that this (optional) extension
   could be useful for debugging.

   [ NOTE: This option adds complexity to implementations, and so we are
   making it optional for recursive servers to implement. ]

6.1.  Option Format

   The option is structured as follows:

                 +0 (MSB)                        +1 (LSB)
      +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
   0: |                         OPTION-CODE                       |
      +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
   2: |                        OPTION-LENGTH                      |
      +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
   4: | D | U | S |             RESERVED                          |
      +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+

   OPTION-CODE  2 octets per [RFC6891].  For Serve-Stale the code is TBD
      by IANA.

   OPTION-LENGTH:  2 octets per [RFC6891].  Contains the length of the
      payload following OPTION-LENGTH, in octets.

   D  Flag - if set, the client explicitly does NOT want stale answers.
      If clear, the client would like additional information.

   U  Flag - this indicates that the server understand Serve-Stale EDNS
      option, and more information is communicated via the S flag [Ed
      note: this exists to get around the issue of some authorative
      servers simply echoing back the ENDS options ]

   S  Flag - if set, this indicates that the answer provided is stale.
      If clear, it indicates that the answer is NOT stale.

   RESERVED  Reserved for future use.  Should be set to zero on send and
      ignored on receipt.

   [ Editor note: Dear WG - we are somewhat shaky on this section, and
   would like feedback on it... ]

7.  Example Method

   There is conceivably more than one way a recursive resolver could
   responsibly implement this resiliency feature while still respecting
   the intent of the TTL as a signal for when data is to be refreshed.

   In this example method three notable timers drive considerations for
   the use of stale data, as follows:

   o  A client response timer, which is the maximum amount of time a
      recursive resolver should allow between the receipt of a
      resolution request and sending its response.

   o  A query resolution timer, which caps the total amount of time a
      recursive resolver spends processing the query.

   o  A maximum stale timer, which caps the amount of time that records
      will be kept past their expiration.

   Recursive resolvers already have the second timer; the first and
   third timers are new concepts for this mechanism.

   When a request is received by the recursive resolver, it SHOULD start
   the client response timer.  This timer is used to avoid client
   timeouts.  It SHOULD be configurable, with a recommended value of 1.8
   seconds.

   The resolver then checks its cache for an unexpired answer.  If it
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
   set the TTL of each expired record in the message to 1 second.  The
   response is then sent to the client while the resolver continues its
   attempt to refresh the data. 1 second was chosen because historically
   0 second TTLs have been problematic for some implementations.  It not
   only sidesteps those potential problems with no practical negative
   consequence, it would also rate limit further queries from any client
   that is honoring the TTL, such as a forwarding resolver.

   The maximum stale timer is used for cache management and is
   independent of the query resolution process.  This timer is
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

8.  Implementation Caveats

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
   trouble refreshing.

   It is important to continue the resolution attempt after the stale
   response has been sent, until the query resolution timeout, because
   some pathological resolutions can take many seconds to succeed as
   they cope with unavailable servers, bad networks, and other problems.
   Stopping the resolution attempt when the response with expired data
   has been sent would mean that answers in these pathological cases
   would never be refreshed.

   Canonical Name (CNAME) records mingled in the expired cache with
   other records at the same owner name can cause surprising results.
   This was observed with an initial implementation in BIND when a
   hostname changed from having an IPv4 Address (A) record to a CNAME.
   The version of BIND being used did not evict other types in the cache
   when a CNAME was received, which in normal operations is not a
   significant issue.  However, after both records expired and the
   authorities became unavailable, the fallback to stale answers
   returned the older A instead of the newer CNAME.

   [ This probably applies to other occluding types, so more thought
   should be given to the overall issue. ]

   Keeping records around after their normal expiration will of course
   cause caches to grow larger than if records were removed at their
   TTL.  Specific guidance on managing cache sizes is outside the scope
   of this document.  Some areas for consideration include whether to
   track the popularity of names in client requests versus evicting by
   maximum age, and whether to provide a feature for manually flushing
   only stale records.

9.  Implementation Status

   [RFC Editor: per RFC 6982 this section should be removed prior to
   publication.]

   The algorithm described in the Section 7 section was originally
   implemented as a patch to BIND 9.7.0.  It has been in production on
   Akamai's production network since 2011, and effectively smoothed over
   transient failures and longer outages that would have resulted in
   major incidents.  The patch was contributed to the Internet Systems
   Consortium and is now distributed with BIND 9.12.

   Unbound has a similar feature for serving stale answers, but it works
   in a very different way by returning whatever cached answer it has
   before trying to refresh expired records.

   Knot Resolver has an demo module here: https://knot-
   resolver.readthedocs.io/en/stable/modules.html#serve-stale

   BIND 9.12 support this functionality, and has some options that can
   be twiddled - these include: stale-answer-enable, stale-answer-ttl,
   max-stale-ttl, The BIND 9.12 announcement says:

   "Akamai contributed a patch, written by a former member of the BIND
   development team, that implements Serve Stale.  Serve Stale returns a
   stale answer when a fresh answer is unavailable due to an
   unresponsive authority.  Akamai has been using a version of BIND with
   this patch internally for several years with good results.  A similar
   approach enabled some public resolvers to continue to offer access to
   popular sites like Twitter during the October 2016 massive DDoS
   against Dyn and we wanted BIND to have the same ability."

10.  new section

   "

11.  Security Considerations

   The most obvious security issue is the increased likelihood of DNSSEC
   validation failures when using stale data because signatures could be
   returned outside their validity period.  This would only be an issue
   if the authoritative servers are unreachable, the only time the
   techniques in this document are used, and thus does not introduce a
   new failure in place of what would have otherwise been success.

   Additionally, bad actors have been known to use DNS caches to keep
   records alive even after their authorities have gone away.  This
   potentially makes that easier, although without introducing a new
   risk.

12.  Privacy Considerations

   This document does not add any practical new privacy issues.

13.  NAT Considerations

   The method described here is not affected by the use of NAT devices.

14.  IANA Considerations

   This document contains no actions for IANA.  This section will be
   removed during conversion into an RFC by the RFC editor.

15.  Acknowledgements

   The authors wish to thank Matti Klock, Mukund Sivaraman, Jean Roy,
   and Jason Moreau for initial review.  Feedback from Robert Edmonds
   and Davey Song has also been incorporated.

16.  References

16.1.  Normative References

   [RFC1035]  Mockapetris, P., "Domain names - implementation and
              specification", STD 13, RFC 1035, DOI 10.17487/RFC1035,
              November 1987, <https://www.rfc-editor.org/info/rfc1035>.

   [RFC2119]  Bradner, S., "Key words for use in RFCs to Indicate
              Requirement Levels", BCP 14, RFC 2119,
              DOI 10.17487/RFC2119, March 1997,
              <https://www.rfc-editor.org/info/rfc2119>.

   [RFC2181]  Elz, R. and R. Bush, "Clarifications to the DNS
              Specification", RFC 2181, DOI 10.17487/RFC2181, July 1997,
              <https://www.rfc-editor.org/info/rfc2181>.

   [RFC6891]  Damas, J., Graff, M., and P. Vixie, "Extension Mechanisms
              for DNS (EDNS(0))", STD 75, RFC 6891,
              DOI 10.17487/RFC6891, April 2013,
              <https://www.rfc-editor.org/info/rfc6891>.

16.2.  Informative References

   [RFC7719]  Hoffman, P., Sullivan, A., and K. Fujiwara, "DNS
              Terminology", RFC 7719, DOI 10.17487/RFC7719, December
              2015, <https://www.rfc-editor.org/info/rfc7719>.

Authors' Addresses

   David C Lawrence
   Akamai Technologies
   150 Broadway
   Cambridge  MA 02142-1054
   USA

   Email: tale@akamai.com


   Warren Kumari
   Google
   1600 Amphitheatre Parkway
   Mountain View  CA 94043
   USA

   Email: warren@kumari.net


   Puneet Sood
   Google

   Email: puneets@google.com

```
