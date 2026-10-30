---
title: "Machine-Readable Training Preference Signal"
abbrev: "Training Preference Signal"
category: std

docname: draft-kelley-aipref-training-preference-signal-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Web and Internet Transport"
workgroup: "AI Preferences"
keyword:
 - robots.txt
 - AI preferences
 - AI training
venue:
  group: "AI Preferences"
  type: "Working Group"
  mail: "ai-control@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/ai-control/"

author:
 -
    fullname: Kevin Kelley
    organization: Anthropic
    email: kelleyk-ietf@kelleyk.net

normative:
  ROBOTS: RFC9309
  VOCAB: I-D.draft-ietf-aipref-vocab-08

informative:
  ATTACH: I-D.ietf-aipref-attach
  HTTP: RFC9110

...

--- abstract

This document describes how a service operator binds usage preferences
(e.g., "no AI training") to paths by writing ordinary `robots.txt`
groups keyed by a per-purpose policy token in the `User-Agent` line.
It introduces no new directive names and no new grammar: the path
language, longest-match precedence, and group structure of `robots.txt`
are not changed.


--- middle

# Introduction {#intro}

A service operator binds usage preferences (e.g., "no AI training") to
paths by writing ordinary `robots.txt` groups keyed by a
per-purpose **policy token** in the `User-Agent` line.

~~~
User-Agent: __usage_ai-training
Disallow: /
Allow: /press-kit/
~~~
{: title="A policy-token group"}

This expresses a preference *against* AI training for all paths
except those under `/press-kit/`, for which it expresses a preference
*in favor* ({{checking}}).

This document introduces no new directive names and no new grammar; the
path language, longest-match precedence, and group structure defined by
{{ROBOTS}} are not changed.  As a result, the wide variety of existing
parsers deployed across the Internet will continue to interpret
traffic management signals as they did previously, and will ignore
policy-token groups as they would any group for a user agent they do
not recognize.

## Goals {#goals}

1. **Path-specific bindings.** Preferences can be expressed per path,
   and not only site-wide ({{checking}}).

2. **Minimal distance from RFC 9309.** Semantics, and ideally
   syntax, stay as close to {{ROBOTS}} as possible, so that authors and
   implementers can use the mental model they already have.

3. **No large-scale duplication.** `robots.txt` files are
   size-constrained ({{Section 2.5 of ROBOTS}}), and some deployed
   files already contain heavily duplicated groups.  Expressing
   preferences does not require restating existing rules
   ({{policy-token}}).

4. **Unchanged legacy behavior.** A parser or consumer that does not
   implement this mechanism behaves as before.  This holds for parsers
   that are actually deployed, not only for idealized RFC 9309
   parsers.  Conversely, implementations treat files that do not use
   this mechanism exactly as before ({{checking}} and
   {{independence}}).

5. **Consistent interpretation across consumers.** This mechanism does
   not depend on the divergent parts of deployed parser behavior
   ({{policy-token}}).


# Conventions and Definitions

{::boilerplate bcp14-tagged}

service operator:
: An entity that operates an HTTP service, such as a website, and
  publishes a `robots.txt` file for it.  For preferences expressed
  through this mechanism, the service operator is the "declaring
  party" ({{Section 2 of VOCAB}}).  {{ROBOTS}} calls this party a
  "service owner".

consumer:
: An entity that interprets preference signals that a service operator
  expresses through the mechanism defined in this document.

parser:
: In the sense of {{ROBOTS}}; software that parses `robots.txt` files
  and matches product tokens and paths against their rules.

policy token:
: A product token ({{Section 2.2.1 of ROBOTS}}) whose RFC 9309 signals
  a consumer uses to determine how content will be used.  A policy
  token need not appear in the `User-Agent` header field
  ({{Section 10.1.5 of HTTP}}) of any outbound request.  This document
  introduces a new, cross-vendor policy token ({{policy-token}}).

proprietary policy token:
: A policy token documented by a single consumer.  Some service
  operators already include these in their `robots.txt` files.


# Vocabulary {#vocabulary}

The `__usage_ai-training` policy token corresponds to the definition
that the IETF aipref Working Group has arrived at
({{Section 4.1 of VOCAB}}).  The following definitions are quoted from
that document:

AI Training:
: Using an asset to modify the learned parameters of a generative AI
  model.

  The training of models that are used to perform exclusively
  non-generative tasks is not included in this category, even if the
  model is capable of generative tasks.

Generative AI model:
: An AI model that is used to generate synthetic content in one or more
  modalities. Alternatively, an AI model that is made available for use
  to generate synthetic content in one or more modalities. This
  definition does not include a model that is used, or made available
  for use, for classification, ranking, or scoring, even where the model
  generates rationale for its output.

This document applies those definitions as follows:

asset:
: The content retrieved from a path.  The preferences that apply to an
  asset are those expressed in the `robots.txt` file in effect when the
  asset was retrieved ({{Section 2.4 of ROBOTS}}; see also {{ATTACH}}).

This document does not define a parallel vocabulary; it tracks the
work of the aipref Working Group.  Implementers SHOULD consider
updating their definitions, and adding or removing vocabulary terms,
as the aipref Working Group reaches consensus in additional areas.


# Design {#design}

## The Policy Token {#policy-token}

This document defines the policy token `__usage_ai-training`.

The policy token conforms to the product-token syntax in
{{Section 2.2.1 of ROBOTS}} (letters, `_`, `-`).

Consumers MUST match policy tokens exactly, apart from case, as for
any product token.  (Note that some deployed parsers match more
flexibly; the `__` prefix makes accidental matches unlikely.)

A single group MAY name several policy tokens (consecutive
`User-Agent` lines; {{Section 2.2 of ROBOTS}}) to bind its rules to
several purposes.  Policy tokens and other product tokens MAY be
mixed in a group.

## Checking for Preferences {#checking}

To determine whether a preference has been expressed for a given path,
a consumer looks up the path and policy-token combination as specified
in {{Sections 2.2.1 and 2.2.2 of ROBOTS}}.

However, wildcard groups (i.e., `User-Agent: *`) MUST NOT be interpreted
as matching the policy token.

If the lookup finds that the path is disallowed, a preference against
the use described by the policy token has been expressed for that path.
If it finds that the path is allowed, a preference in favor of that use
has been expressed.

If no group contains the `__usage_ai-training` policy token (i.e.,
there is no `User-Agent: __usage_ai-training` line in the file), then
no preference has been expressed.  If such a file contains a
proprietary policy token, consumers SHOULD interpret that token as
they would in the absence of this document, but MUST NOT treat it as
expressing a preference in the context of this document (see
{{precedence}}).

## Independence from Crawler Rules {#independence}

Consumers MUST interpret the RFC 9309 signals being expressed based on
their own product token(s) exactly as they would in the absence of
this document.  That is, `ExampleBot` still checks whether there are
rules for `ExampleBot`; if there are, they still express the service
operator's preference about crawling.  Nothing in this document
changes that signaling.

Content-usage preferences and traffic management signals can both be
expressed in `robots.txt`, but they are independent and orthogonal.

## Effects on Crawling Behavior {#crawling}

This policy token is not a directive for crawling behavior.

A consumer MAY use preferences to inform crawling.  For example, if it
crawls only for purposes covered by the policy token, it MAY refrain
from retrieving resources for which a preference against those
purposes has been expressed.

If a service operator wishes to limit crawling, it MAY use groups for
crawlers' product tokens in `robots.txt` to do so.

## Precedence {#precedence}

Specific expressions SHOULD take precedence over general ones.

Consumers MAY document that they interpret rules that match
proprietary policy tokens as being more specific than the
`__usage_ai-training` policy token.

Private agreements MAY override preferences expressed via this
mechanism.


# Security Considerations {#security}

The security considerations of {{Section 3 of ROBOTS}} apply to this
mechanism.


# Privacy Considerations {#privacy}

This document does not introduce any new privacy considerations.


# IANA Considerations {#iana}

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

The author is grateful to the Working Group's chairs, Mark Nottingham
(Cloudflare) and Suresh Krishnan (Cisco), to the editors of the
vocabulary draft, Martin Thomson (Mozilla) and Paul Keller (Open
Future), and to the responsible Area Director, Mike Bishop (Akamai),
whose incredible effort to facilitate discussion and endless patience
(in addition to their own individual contributions) have made the
Working Group's progress possible.

The author also thanks the following people for their substantial
contributions to shaping the Working Group's discussion and for their
partnership.  Affiliations are listed for identification only, since
IETF participants contribute as individuals.

| Name                 | Affiliation                                   |
|----------------------|-----------------------------------------------|
| Farzaneh Badiei      | Digital Medusa                                |
| Lila Bailey          | Internet Archive                              |
| Kevin Bankston       | Center for Democracy and Technology           |
| Fabrice Canel        | Microsoft                                     |
| Alissa Cooper        | Knight-Georgetown Institute                   |
| Sonia Cooper         | Microsoft                                     |
| Glenn Deen           | Comcast NBCUniversal                          |
| Caleb Donaldson      | Google                                        |
| Daniel Etcovitch     | Anthropic                                     |
| Christopher Flammang | Elsevier                                      |
| Max Gendler          | News Corp                                     |
| Alex Gibson          | News Corp                                     |
| Jordan Gimbel        | Microsoft                                     |
| Mark Gray            | OpenAI                                        |
| Nate Hake            | Travel Lemming                                |
| Gary Illyes          | Google                                        |
| Meredith Jacob       | American University Washington College of Law |
| Mirja Kühlewind      | Ericsson                                      |
| Laurent Le Meur      | EDRLab                                        |
| Jo Levy              | Norton Law                                    |
| Greg Lindahl         | Common Crawl Foundation                       |
| Krishna Madhavan     | Microsoft                                     |
| Sarah McKenna        | Sequentum                                     |
| Angela Mills Wade    | European Publishers Council                   |
| John Mueller         | Google                                        |
| Chris Needham        | BBC                                           |
| Elaine Newton        | Amazon Web Services                           |
| Victoria Noble       | Electronic Frontier Foundation                |
| Sebastian Posth      | Liccium                                       |
| Brendan Quinn        | IPTC                                          |
| Courtney Radsch      | Open Markets Institute                        |
| Felix Reda           | GitHub                                        |
| Eric Rescorla        | Independent                                   |
| Matt Rogerson        | Financial Times                               |
| Leah Romm            | Cloudflare                                    |
| Leonard Rosenthol    | Adobe                                         |
| Malikat Rufai        | Meta                                          |
| Achim Schlosser      | Bertelsmann                                   |
| Rony Shalit          | Bright Data                                   |
| Bradley Silver       | Advance                                       |
| Erin Simon           | Google                                        |
| Esther Tetruashvily  | OpenAI                                        |
| Michael Tremante     | Cloudflare                                    |
| Thom Vaughan         | Common Crawl Foundation                       |
| James Whymark        | Meta                                          |
| Timid Robot Zehta    | Creative Commons                              |

Many more people shaped the Working Group's discussion than appear
above, and this list has almost certainly left out someone who belongs
on it.  Any such omission is inadvertent, and the author apologizes in
advance.
