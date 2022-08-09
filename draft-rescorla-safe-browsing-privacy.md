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














# Alternative Designs

## Proxies

## PIR

# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Hash Length Selection {#hash-size}

[TODO}

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
