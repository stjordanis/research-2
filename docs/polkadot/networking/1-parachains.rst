\====================================================================

**Authors**: Ximin Luo

**Created**: 2020-05-08

\====================================================================

====================
Parachain networking
====================

TODO: this document will discuss the networking interface between parachains
and the relay chain.

Background
==========

- Collators need to send PoV block information to their parachain validators.

- Ids of parachain validators are on-chain and public knowledge

- Collators are in general unauthenticated

  - In future, the parachain validation function may supply an interface that
    allows the parachain to restrict its collator membership set, this will
    take a while to develop.

- The task is to distribute PoV block data from various entities to parachain
  validators. Some of these entities may be malicious, some may be actual
  collators.

- Blocks are unauthenticated during distribution, however after distribution
  once we've received it fully, we can run the validation function on it to
  authenticate it.

  In the meantime, we have to buffer potentially bad data from malicious nodes,
  but we can remember this information and prefer peers with good history.

  - In future, the parachain validation function may supply an interface that
    allows the parachain to specify a way to pre-validate a block, e.g. that it
    must be signed by specific keys.

- Load-balancing across the parachain validators: they can each track capacity
  and redirect collators to other parachain validators.

First principles
================

We want collators to send us PoV blocks in a way that uses our bandwidth as
efficently as possible. We want to make it hard for attackers to significantly
reduce the normal operating efficiency.

Attackers can:

- compete with connection attempts, by regenerating a new identity
- compete with bandwidth, e.g. by sending valid but redundant / old data. This
  attack is relatively costly to pull off, however.

We can:

- control per-peer bandwidth
- select which peers we accept connections from. (typically we cannot directly
  select, but we can accept a connection then immediately close it.)

During a validator-group rollover, we would like to pass on reputation
information from the old group to the new group, so that we can more quickly
find good peers.

Passing "reputation" between validators also allows us to perform some
load-balancing, since we are aware of which validators are processing which
collators.

High-level ideas:

1. Track the efficiency of each peer, i.e. byte counts for validation-pending,
   valid vs valid-but-redundant, vs total byte counts.

2. Optional: top efficient peers get reserved more bandwidth.

3. If any peer remains in the bottom (say) 1/5 efficiency for > X time, we drop
   them and swap them out for a new stranger peer, if one is waiting.

4. If any peer sends us data that is eventually invalidated, they get put on
   a blacklist. Since identities are easy to regenerate, we record their
   address as well, thus building up a "heat map" of bad IP addresses. Since IP
   addresses are dynamic, this heat map should gradually fade over time.

5. When selecting new stranger peers, we prefer addresses that are on the
   cooler part(s) of the "heat map".

   Determining which IP addresses are "close" to each other is hard; but since
   addresses are allocated by block, a basic strategy is better than nothing.

   There may be other criteria we can use, IP address is the most obvious.

   TODO: research how to actually accomplish this; accept() does not expose to
   you the full set of peers that are currently waiting to connect to you.

6. When rotating groups, we tell the new validators which peers we think are
   best, i.e. their identities. They are then free to prefer these peers.

   In other words, we have a whitelist over identities, which is passed around
   here, and a blacklist over addresses, which (in this idea) is not passed
   around.

We track efficiency and not just validity, which there are a whole class of
sophisticated bandwidth-wasting attacks that transmit valid-but-redundant data.
This is a straightforward way of making these attacks much harder, since the
attacker is forced to compete with actual genuine peers with regards to the end
performance that the application actually cares about.

Current thoughts
================

- Block data may be split, so that we can receive different pieces from
  multiple peers at once. This can be done e.g. by a merkle tree, with the root
  hash being the identifier of the block. (As per the previous point, this root
  hash would be unauthenticated during distribution).

- Parachain validators should be connected to each other, to help with
  distribution.

- TODO: having the pre-validation interface (as described above) soon, would be
  a great help. Reputation systems are an open research area; relying on this
  in production is a risk. For PoW and other similar chains, a pre-validation
  interface is sufficient for anti-DoS security on the networking layer, and is
  much easier to analyse than a reputation system.

Reputation systems overview
===========================

The term "reputation system" is commonly thrown around to refer to a few
different things:

1.  Aggregating lots of local scores by many sources about some target, into a
    single score for the target. (The aggregated score could be different from
    each source's perspective.) Examples: trust metrics (advogato), PageRank,
    sybil-detection algorithms.

    Polkadot: for parachain networking this is not a major concern since there
    are only 10 parachain validators per parachain. Any aggregation system only
    needs to not be trivially-attackable by 9 other validators. It is relevant
    for chains that want to serve lots of unauthenticated users however, such
    as the relay chain itself, so it's a topic to cover elsewhere.

    For more details see appendix, for now we skip.

2.  In (1) we didn't explain where the local scores come from. Some proposals
    have this manually input by the user, but this is inconvenient & hard to
    reason about. Other systems propose ways to automatically calculate such
    scores, based on empirical observations by the source of that target's
    behaviour.

    Polkadot has currently an ad-hoc implementation of such a system. It is not
    documented and its design decisions are unclear. For parachain networking
    we would like to derive a new system from first principles.

3.  How a source responds to future interactions with a target, depending on
    the score, either aggregate (1) or local (2) or both. This is often grouped
    together with either (1) or (2), but may be better considered separately.

    Specifically, certain responses are inherently unsafe regardless of how you
    arrived at the score driving the response. For example, it is ineffective
    to disconnect or ban a peer with low score, if all they need to do is to
    generate a new public key and IPv6 address, then reconnect you and spam
    your bandwidth again. To protect against this, you must have fine-grained
    control over your resources, and perhaps other mechanisms too.

In this rest of this document we will focus primarily on (2) and (3).

Local scoring
-------------

- Ensure objective source observations are stored, not just the conclusions
  derived from it.

  e.g. store "I saw peer A send me data X" not just "I gave peer A score 3"

  Reason: Past observations are not going to change. But if we change the
  derivation algorithm, we may want to re-derive the score from observations.

  Sometimes we cannot derive a score straight away, e.g. if the derivation
  requires other data we don't have yet. In such a case we will need to defer
  the score derivation, and record this fact as a "debt" so the peer can't
  overwhelm us with deferrable score derivations.

- Tracking provenance.

  TODO: We have our directly-connected collators as neighbours, but we also
  have other parachain validators as neighbours who are gossiping to us pieces
  they've received from other collators. Pieces therefore should be signed by
  each collator - by their transport key; since we're not assuming that the
  parachain requires its collators to have identity keys.

  How to track it in a secure way, what to do with the information.

Responding
----------

- Disconnection, obviously.

- If we only keep the top-scoring peers, this will cause the selection to tend
  to the same set of collators, which may be undesirable. So perhaps we should
  reserve a fraction of our resources for random stranger collators to connect
  to us, to allow new users a chance to participate.

  OTOH the parachain validators get rotated fairly regularly so perhaps this is
  a non-issue and we don't need to come up with a workaround.

Appendix
========

Aggregating scores
------------------

The standard "universal attack" that everyone tries to defend against, is where
the attacker copies the entire topology of the genuine network, and somehow
gets a bunch of genuine nodes to peer with some of their nodes. Solutions must
break the symmetry here by assuming the source peer (doing the aggregation) is
honest, perhaps in addition to certain other user-specified nodes. Then the
aggregated score depends on these seeds of trust.

Because of this attack, solutions without a concept of trust-seeds can be
dismissed out-of-hand as being inherently insecure; Google themselves had to
add this concept into PageRank a few years after they started.
https://www.seobythesea.com/2018/04/pagerank-updated/

State-of-the-art in 2020 is generally based on random-walks / network flow
which work under the assumption that it is costly for an attacker to create
edges to genuine nodes. These algorithms are closely related to community
detection algorithms in network analysis. Some of them propose to be used on
real-world data such as social graphs. In addition to privacy concerns, we
suspect they may generate false positives when the network is genuinely divided
into subcommunities with low flow between them. However there is insufficient
research in this area currently to draw firm conclusions.

Google keep claiming they have internal work beyond PageRank, but refuse to say
publicly what it is or the ideas behind it. Possibly security by obscurity,
possibly genuinely novel & useful stuff they should publish.
