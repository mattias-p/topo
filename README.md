# Topo

## Notation

This algorithm is defined in terms of a [Coloured Petri net] (CPN) with a few
modifications to its rules.
1. When a variable is used in a single place and never referred to, its name is
   replaced with a placeholder underscore.

2. For double ended arrows, the arrow head pointing to a place is depicted as
   empty.
   This is meant to let the reader know which arrows coming into a place are
   double ended and which ones are not.
   Even when the arrows themselves are very long.

3. The guard expression of a transition may use a function named parent.
   The `parent(FQDN: domain) → FQDN` function returns a copy of `domain` with
   its least significant label removed. If `domain` contains no labels, an
   identical FQDN is returned.

4. In addition to the usual kind of places there is also "request places" and
   "response places".

   When a token is created in a "request place", a DNS request is sent to the
   specified by the token.
   The token also specifies the qname.
   The qtype is taken from the name of the place.

   When a response is received it is interpreted according to a set of
   patterns.
   Each time a pattern is matched, a token is created in the associated
   response place, given that such a place exists.
   The following patterns are defined:
    1. Nodata
       Criterion: A token is created in this place if the response has the
       NODATA pseudo RCODE.
    2. Ans
       Criterion: A token is created in this place for each authoritative answer
       record for the qname of the qtype.
    3. AnsGlue
       Precondition: A place of this kind must only be attached to request
       places whose qname is NS.
       Criterion: A token is created in this place for each A record and each
       AAAA record in the additional section whose owner name matches the
       NSDNAME of an authoritative NS answer for the qname of the qtype.
    4. Ref
       Criterion: A token is created in this place for each NS record in the
       authority section.
    5. RefGlue
       Criterion: A token is created in this place for each A record and each
       AAAA record in the additional section whose owner name matches the
       NSDNAME of an NS record in the authority section.

   The color set of a request place must be ADDR ⨉ FQDN: server, qname.
   The exact color set of a response place is defined by the place itself, but
   it must be a tuple consisting of the following elements:
    1. ADDR: server - the IP address of the server the request was sent to.
    2. FQDN: qname - the QNAME used in the request.
    3. ADDR: glue - the ADDRESS of the A or AAAA record. Only valid for AnsGlue
       and RefGlue response places.
    4. FQDN: nsdname - the NSDNAME of the NS record. Only valid for Ref and
       RefGlue response places, as well as Ans and AnsGlue response places
       provided that the qtype is NS.
    5. ADDR: address - the ADDRESS of the A or AAAA answer. Only valid for
       Ans and AnsGlue response places provided that the qtype is A or AAAA.

   The associations between request places and response places are depicted
   with dotted arrows.

   A label may be attached to the dotted arrow indicating the number of tokens
   that may be created in the response place as a result of a single request.
   The minimum number is always zero.
   The maxmimum number is either 1 or positive infinity depending on the kind
   and color set of the response place.
   If the label is not present the maximum can still be inferred, but adding
   the label is helpful to the reader.

5. Unlike in regular CPNs where places contain multisets of tokens, these places
   contain sets of tokens.

6. For each qtype specified by at least one request place, there is a tracker
   that keeps track of all tokens that have been created in request places with
   that qtype.

7. A transition can only fire if it would create at least one token in a place
   where an identical token hasn't been before.

8. A transition can only fire if it would not create a token in a request place 
   that would be recognized by the tracker of that place.

9. Execution continues with enabled transitions firing in any order until all
   transitions are dead, causing the execution to terminate.

Rules 5, 6, 7 and 8 are backwards compatible with [prioritised petri nets], 
but by using these rules instead, we can keep the diagram more succinct.

#### Inputs
* A set of root hints.
* A domain name.

#### Outputs
* A set of chains of zone authorities. The chains of zone authorities for the
  given domain and for each encountered out-of-bailiwick name server.
* A set of (domain name, IP address) pairs. The domain name and address of each
  encountered out-of-bailiwick name server.

#### Procedure

![diagram](topo.png)

The Target and Hint places are where the input objects are added before the
starting the execution.
All other places start out empty.

The t<sub>1</sub> transition initializes the net from the inputs by putting an
object for the target domain into the Domain place and one object per root
server into the Auth place.

The Domain place tracks what domains we're interested in.
It represents all the nodes in a DNS tree where the leaves are either a target
given in the inputs or the name of an out-of-bailiwick name server.

The first thing to happen after t<sub>1</sub> is that t<sub>2</sub> makes sure
all ancestors of the added domain are also added.

The Auth place tracks what servers are authoritative for each domain we're
interested in.

Recall that after initialization t<sub>2</sub> starts to populate Domain with
the ancestors of the target domain.
Once the top level domain of the target domain is added to Domain, t<sub>3</sub>
wakes up and starts adding objects to SOA/TX, one for each root server.

The t<sub>4</sub> transition sends out each of the queries, and when each
response is received, t<sub>5</sub> is fired.
If a response is NODATA or if it contains an authoritative SOA record for the
queried name, the queried server is added to Auth as authoritative for the
queried name.
For any referral with glue, the glue address is added to Auth as authoritative
for the queried name.
For any out-of-bailiwick referral, the NSDNAME is added to Oob together with the
name it is authoritative for.

As servers for the top level domain are added to Auth, t<sub>3</sub> will start
waking up adding queries about the second level domain to those servers.
And so on until the servers for the target domain have all been found, and
execution terminates.

Unless, of course, out-of-bailiwick name servers were found along the way.
In that case we want to add the addresses for the name server to the Auth place.
If we can do that the machinery described above will take care of the rest on
its own.

We we need to do is to determine the authoritative servers for its domain name,
ask those servers about for addresses, and we need to connect the addresses back
to the domain name that gave us the out-of-bailiwick name server.

By adding the name server name of the out-of-bailiwick name server to Domain
using t<sub>6</sub>, we ensure that the servers that are authoritative for that
name will be found.

When servers are added to Auth that are authoritative for the a name server name
in Oob (or the other way around), t<sub>7</sub> wakes up and starts adding
objects to A/TX and AAAA/TX.

The transitions t<sub>8</sub>, t<sub>9</sub>, t<sub>10</sub> and t<sub>11</sub>
work together to query the servers for A nad AAAA records and those records are
added to Addr.

Once addresses start showing up in Addr, t<sub>12</sub> will start waking up to
correlate out-of-bailiwick referrals with name server addresses and adding them
to Auth, which is all we needed to do.

#### Discussion

* Auth is not "complete" when NXDOMAIN is returned for a domain in the middle of
  Domain.
* Oob is defined in relation to the apex, but this algorithm works as if the
  apex of a domain is always the domain itself.

[Coloured Petri net]: https://en.wikipedia.org/wiki/Coloured_Petri_net
[Prioritised Petri net]: https://en.wikipedia.org/wiki/Prioritised_Petri_net
