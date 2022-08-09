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

The Google Safe Browsing service is used by browsers to determine when
Web sites are potentially malicious, for instance hosting malware or
being phishing sites. The current design of the Safe Browsing protocol
allows the service operator to learn some information about which
sites the user is visiting. This document describes a design which
has improved privacy properties.


--- middle

# Introduction

At a high level, Safe Browsing works by having a list of blocked
strings (domain names and URL prefixes) on the Safe Browsing server.
Prior to visiting a site, the browser checks that the URL and its
components are not on the list; if present, the browser generates
an error and aborts navigation. Although the current API {{SB}} exposes
an endpoint that allows for checking a single string, browsers
do not check this API because doing so would leak a user's
browsing history, as the service could build a longitudinal list
of queries for each IP address.

Instead, the current protocol uses a protocol that provides partial
Private Information Retrieval (PIR). At a high level, this protocol
works as follows:

- The server computes a hash of each string and provides the client
  with a list of unique hash *prefixes*. Typically these prefixes are
  32 bits long. The clients periodically update the prefix list.

- The client computes the hashes of each string and determines
  the intersection of the prefixes of its hash prefixes and those
  provided by the server. Any non-matching hashes are
  OK to connect to.

- The client sends the set of matching prefixes to the server,
  which responds with the set of full hashes matching each
  prefix.

- The client then compares the full (256-bit)  hashes it received to those
  that it computed and blocks and URLs corresponding to matching
  prefixes.

This design is intended to minimize the number of times in which
the client needs to contact the server in order to resolve the
status of the server.

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

Because the client retrieves the relevant hash blocks for each
prefix, these are inherently more timely. However, in principle
if the client caches the result, this can still result in
incorrect results, for instance, a false negative
if prefix X is retrieved and then X || Y is added,
or, a false positive if X is retrieved, yields X || Y,
and then X || Y is removed.


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
identifier; this allows for the detection of abusivy
behavior, but also permits linkability during the lifetime
of the identifier.


## PIR

An alternative approach is to perform each request over
a Private Information Retrieval (PIR) protocol. Unfortunately,
effective PIR protocols have much higher bandwidth and computational
costs than the existing design. [TODO: Wood]


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
rate 2^-60) is enough against a innocuous attacks. However, an attacker
might be able to search the ~2^60 bit space to find a colliding hash
and then arrange for that URL to host malware or phishing content,
thus making a given site unvisitable. This suggests that we want
something more like 128 bits, which requires >2^100 effort to find
a value which hashes to a database entry.

# Acknowledgments
{:numbered="false"}

