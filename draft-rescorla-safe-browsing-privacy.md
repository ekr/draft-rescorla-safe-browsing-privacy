---
###
# Internet-Draft Markdown Template
#
# Rename this file from draft-todo-yourname-protocol.md to get started.
# Draft name format is "draft-<yourname>-<workgroup>-<name>.md".
#
# For initial setup, you only need to edit the first block of fields.
# Only "title" needs to be changed; delete "abbrev" if your title is short.
# Any other content can be edited, but be careful not to introduce errors.
# Some fields will be set automatically during setup if they are unchanged.
#
# Don't include "-00" or "-latest" in the filename.
# Labels in the form draft-<yourname>-<workgroup>-<name>-latest are used by
# the tools to refer to the current version; see "docname" for example.
#
# This template uses kramdown-rfc: https://github.com/cabo/kramdown-rfc
# You can replace the entire file if you prefer a different format.
# Change the file extension to match the format (.xml for XML, etc...)
#
###
title: "Privacy for Safe Browsing"
category: info

docname: draft-rescorla-safe-browsing-privacy-latest
submissiontype: IETF  # also: "independent", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: AREA
workgroup: WG Working Group
keyword:
venue:
  group: WG
  type: Working Group
  mail: WG@example.com
  arch: https://example.com/WG
  github: USER/REPO
  latest: https://example.com/LATEST

author:
 -
    fullname: Eric Rescorla
    organization: Mozilla
    email: ekr@example.com

normative:

informative:
  SB:
       title: "Safe Browsing APIs (v4)"
       author:
       -
         ins: Google
       target: https://developers.google.com/safe-browsing/v4

--- abstract

The Google Safe Browsing service is used by browsers to identify
potentially malicious web sites, for example, sites that distribute
malware or that are part of a phishing campaign. Browsers may warn users
before allowing navigation to a site that is flagged by the Safe
Browsing service, or take other steps to reduce the chance that a user
is harmed by a malicious site. The current design of the Safe Browsing
protocol allows the service operator to learn some information about
which sites the user is visiting. This document describes a design which
has improved privacy properties.


--- middle

# Introduction

At a high level, Safe Browsing works by having a list of blocked
strings (domain names and URL prefixes) on the Safe Browsing server.
Prior to visiting a site, the browser checks that the URL and its
components are not on the list; if the URL is on the list, the browser generates
an error and aborts navigation. Although the current API {{SB}} exposes
an endpoint that allows for checking a single string, browsers
do not use this API because doing so would leak a user's
browsing history, as the service could build a longitudinal list
of queries for each IP address.

Instead, the current protocol uses a protocol that provides partial
Private Information Retrieval (PIR). At a high level, this protocol
works as follows:

- The server computes a hash of each blocked string and provides the client
  with a list of unique hash *prefixes*. Typically these prefixes are
  32 bits long. The clients periodically update the prefix list.

- The client computes the hashes of each string to be queried in the
  Safe Browsing list, and determines whether the prefix of any of those
  hashes appears on the list provided by the Safe Browsing server. Any
  hash prefixes that do not appear on the server's list are OK to connect
  to.

- The client sends the set of prefixes that did appear on the list to the server,
  which responds with the set of full hashes matching each
  prefix.

- The client then determines whether the full (256-bit) hashes of its
  query strings appear in the set of full hashes received from the server.
  Any that match are treated as flagged by the Safe Browsing service.

This design reduces the number of times that clients must contact the
server in order to determine the Safe Browsing status of a site, while
also avoiding the cost of keeping a full copy of the list of blocked
strings at each client.

## Privacy Issues

There are on the order of a million (2^20) strings in the database and
there are 2^32 possible hash prefixes, so approximately 2^-12 (1/4096)
hash prefixes corresponds to a blocked entry. This means that about
1/4096 strings will require a database lookup. Naively, this would mean
that the server would obtain information about 1/4096 of the URLs
that a user visits, but the number is probably closer to 1/1000
because the client queries for prefixes of the URL as well as
the whole URL; the precise number varies depending on the number
of components in each URL. Moreover, the server can opt to receive
information about given sites by spuriously publishing their
hash prefixes. A full privacy analysis of Safe Browsing is out of scope for
this document, but it should be clear that these properties
are not ideal.

## Timeliness Issues

In addition to the privacy issues discussed in the previous section,
the current Safe Browsing design has suboptimal timeliness properties.
Specifically, because the client periodically updates the hash
prefix list, and does not check with the server for non-matching
hashes, it is possible to have false negative results if the
hash was added during the update interval. Because phishing
attacks are often of quite short duration, this has a significant
impact on the effectiveness of Safe Browsing.

Because the client retrieves the full-length hashes for each
matched prefix, these results are inherently more timely. However, in principle
if the client caches the full-length hashes, this can still manifest
incorrect results, for instance, a false negative
if prefix X is retrieved and then X || Y is added,
or, a false positive if X is retrieved, yields X || Y,
and then X || Y is removed.

# Measuring Safe Browsing Privacy

Ideally, safe browsing queries could be served in a way that provides
information-theoretic security, meaning no information at all about
browsing history is revealed to the server, or in a way that provides
computational security, meaning a computationally-bounded server cannot
learn anything about the browsing history. However, it may be
significantly easier to implement a scheme in which the server learns a
bounded, small amount of information about the browsing history.
Deciding whether a given query reveals a "small" amount of information
is difficult. Learning specific sites that a user has or is very likely
to have visited, is probably more than a "small" amount of information.
Being able to infer probabilities of the user having visited sites that
are close, but not identical, to the probabilities of an average user
having visited those sites, might be considered a "small" amount of
infomation. Differential privacy [cite] provides one framework for
assessing how much information is revealed. [Toledo et al.] analyze
several PIR schemes in the framework of differential privacy. However,
while differential privacy provides a framework for analysis, it does
not establish the amount of information leakage that is acceptable in
any particular application.

[TODO: Discuss epsilon and delta, and the impact of zero-probability
events on achievable values thereof?]

If the probability that a random internet user has visited a particular
site is $10^{-6}$, and the probability that a user who has made a certain
query to the safe browsing server has visited that site is $10^{-4}$, is
that a meaningful loss of privacy? What if the probability that
a random internet user has visited a particular site is $10^{-2}$, and that
increases to 0.99 for users who have made a particular safe browsing
query?

The existing "dummy requests" scheme implemented in Firefox for safe
browsing queries is subject to this kind of distortion. Safe browsing
records associated with very popular sites will be requested much more
frequently due to actual navigation to the popular site, than they will
due to random selection for a dummy request. An open question is the
extent to which any amount of "plausible deniability" is sufficient.
That is, if there is any chance, no matter how small, that a particular
safe browsing lookup does not indicate actual browsing activity, does
that provide sufficient privacy for safe browsing requests?

[De Blasio] describes a scheme for SCT auditing in Chrome that provides
a level of privacy by requesting blocks of SCT hashes. It is said to be
providing k-anonymity, but in the strictest sense of the term that may
not be true -- there are a total of k records that could be the record
of interest, but there are not necessarily k users making the same
set of requests.

# An Improved Design

The privacy issues described in {{privacy-issues}} result from the
client needing to contact the server to verify the hash, which is a
consequence of the 32-bit hash prefix being too short to provide
precise results. Because the client only contacts the server for
matching prefixes, this leaks browsing history.

This is easily addressed by having the server provide longer hashes,
on the order of 128 bits (see {{hash-size}}). This provides an
immediate improvement in privacy because the client can immediately
decide the status of any URL, thus avoiding any URL-dependent queries
to the server. In addition, it provides a performance benefit because
the client does not need to round-trip to the server.

## Timeliness

A strict "full hash" design has inferior timeliness properties.

Specifically, because the client need not confirm matches with
the server, any hashes which are incorrectly added to the list--or
need to be removed for some other reason--remain until the next
time the client updates; this creates false positives. By
contrast, with the current design the client makes its final
blocklist checks in real time.

This design is also slightly worse for false negatives. Consider
the following timeline:

~~~
T0      Database contains X || Y
T1      Client downloads database
T2      X || Z added to database
T3      Client attempts to go to X || Z
~~~

With the current design, the client will check with the server
at time T3 and thus will refuse to visit the site. By contrast,
with a full hash version, the client will check the database
at T3, find only X || Y, and continue.

Both of these issues can be addressed by having the client
receive more frequent updates from the server. It may also
be useful for the server to have a mechanism to publish
high prioirity updates, for instance by having the client
poll for them in real-time or by having a publish/subscribe
type notification channel.

It is important to note that because none of the client
actions in this scenario depend on the browsing history,
it is safe to do them all on a single communication channel,
even though this allows them to be linked. This is a
potential issue with alternative designs based on proxies,
as described in {{proxies}}.

## Bandwidth Comparison

[TODO: how does this compare to SB now?]

# Alternative Designs

There are two main alternative designs for preventing the
server from learning the client's queries.

## Proxies

The obvious approach is to simply use a proxy (e.g.,
OHAI
{{?I-D.ietf-ohai-ohttp}} between the client and the
server, thus concealing the client's IP address. This
provides good privacy as long as the server and proxy
do not collude.

However, the queries need to be unlinkable in order to prevent
situations in which the server determines the client's
identity from part of its browsing history and then is
able to link that to some incriminating part of the user's
activity. This is a particular problem in the context
of proposals like Safe Browsing v5 in which the client
performs real-time checks for most URLs (filtered by
an allow list), and thus the server sees more of the
user's history.

This requirement for unlinkability makes the use of
connection-oriented proxies such as MASQUE {{?I-D.ietf-masque-connect-udp}}
problematic, as a new connection must be initiated for
each request, which introduces latency. In addition,
it creates problems for anti-DoS mechanisms which
depend on giving clients a short-term persistent
identifier; this allows for the detection of abusive
behavior, but also permits linkability during the lifetime
of the identifier.


## PIR

An alternative approach is to perform each request over
a Private Information Retrieval (PIR) protocol. Unfortunately,
effective PIR protocols have much higher bandwidth and computational
costs than the existing design. [TODO: Wood]

Table 2 in [CG19] provides an overview of known PIR protocols for both
the single-server and two-server cases. The best known protocols have
$O(\sqrt{n})$ online server computation, but require amortizing offline
(preparatory) computation over many queries to achieve that. Depending
on the rate of change of the safe browsing database, these schemes may
not be applicable. [MZRA22] adapts the [CG19] scheme for dynamic
databases, but in a way that does not apply to PIR-by-keywords. For the
fully online case, [GI14] has $O(n)$ online computation and $O(\log n)$
communication.

The techniques of [BIM04] can trade reduced online server computation
for increased server storage. Given the modest size of the safe browsing
database, these schemes may be useful.

A hybrid approach like the following may be viable for queries to a safe
browsing database with $O(10^6)$ records:

* The safe browsing database is divided into $O(250)$ buckets. Most sites
  are assigned to one of these buckets using a hash. Each lookup leaks
  which bucket it is searching. As long as sufficiently many and varied
  sites are assigned to each bucket, this leakage may be deemed acceptable.
  Popular sites are replicated in every bucket, and may be looked up in
  any bucket at the client's discretion, thus leaking nothing.

* Within each bucket, a PIR scheme is used to search the $O(4 \times 10^3)$
  records assigned to that bucket.

Based on data that most phishing sites are active for less than 10
minutes [Cite private specification], it is assumed that offline-online
PIR schemes do not apply well for this purpose. However, a more thorough
analysis of the safe browsing database would be informative. If the safe
browsing database can be divided into short- and long-term subsets, and
the long-term subset makes up the bulk of the data, then using
offline-online PIR for the long-term data might provide meaningful gains
in efficiency.

# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Hash Length Selection {#hash-size}

The false positive rate of a system with a b-bit hash is effectively
2^{-(b-20)}. This suggests that an 80 bit hash (false positive
rate 2^-60) is sufficient assuming only random collisions. However, an attacker
might be able to search the ~2^60 bit space to find a colliding hash
and then arrange for that URL to host malware or phishing content,
thus making a given site unvisitable. This suggests that we want
something more like 128 bits, which requires >2^100 effort to find
a value which hashes to a database entry.

# Acknowledgments
{:numbered="false"}

