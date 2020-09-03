\====================================================================

**Authors**: Ximin Luo, Rob Habermeier, Fatemeh Shirazi

**Last updated**: 2020-06-04

\====================================================================

====
XCMP
====

This subprotocol runs whenever the relay chain block production protocol has output a new candidate block, similar to the A&V protocol.

This candidate block references a bunch of parachain blocks, that might indicate that those parachains wish to send messages to other parachains. To save bandwidth, and since no other parachains need to receive the data, the message bodies are not contained within these blocks, and must be transferred separately. The purpose of this subprotocol is to do that.

Background
==========

XCMP high-level overview
------------------------

XCMP is designed to achieve ordered, reliable, and fair delivery.

Terminology note: for the logic relevant to this document, all the messages for a given (sender, block) are processed in a single batch by the recipient, so from here on we will refer to "the" message at a given (sender, block) even though in practise this will consist of multiple smaller physical messages. Every logical message is authenticated as a signed merkle tree root (or equivalent) and this merkle root is stored in the relay chain.

Every parachain block may contain messages to other parachains, indicated by the presence of merkle roots in the block body, one per recipient. As just mentioned, the message bodies are not stored with the block. A message is considered "sent" at relay-chain block ``B``, if the parachain block referencing that message is included in or before ``B``.

Every parachain has a logical egress queue, indicating outgoing messages that have not yet been acknowledged by the recipient parachain. These queues are compactly represented in the relay-chain state at each block. The representation is maintained by trusted relay-chain code, using data from parachain block data. This allows senders to know which messages bodies they should prepare to send.

Every parachain also has a logical ingress queue, indicating incoming messages that they have not yet acknowledged themselves. These queues are also compactly represented within the relay-chain state at each block. The representation is maintained by trusted relay-chain code, using the egress queue representations as well as acknowledgements from recipients (see below). This allows receivers to know which message bodies they should prepare to receive.

**Ordering**. For a given sender, its egress queue may be thought of as several per-recipient egress queues, whose ordering is independent of each other. That is, there is no guarantee about the relative ordering of when messages will be delivered to recipients between multiple egress queues. By contrast, for a given recipient, its ingress queue is totally-ordered across all recipients - messages queued from multiple senders at a given relay-chain height, are placed in the ingress queue in a pre-determined order to ensure fairness.

To ensure reliability, every parachain block building on relay-chain block ``B`` must include at least one new acknowledgement of a message in their inbox at block ``B``, if it is non-empty. Acknowledgements are transitive, i.e. an ack of the message from sender chain ``C`` at block ``B``, implies an ack of any messages from the same sender at every ancestor of ``B``, i.e. ``B~``, ``B~2``, etc, to use ``gitrevisions(1)`` syntax. To ensure ordering and fairness, recipients are not allowed to ack messages from a block ``B``, until all incoming messages from blocks ``B~``, ``B~2``, etc have been acked.

These transitive acknowledgements are stored on the relay chain as a *watermark*, for each recipient parachain. This consists of some relay-chain block ``W~`` for which the recipient has acknowledged all previously-sent messages, and a set of sending parachains for block ``W`` for which the recipient has acknowledged messages (sent in block ``W``). This data is used to help calculate and maintain the aforementioned ingress and egress queues.

**Bounds**. Ingress queues are bounded in size, and since they are derived from the egress queues, this implies that the egress queues are also bounded in size. That is, if a recipient does not "consume" its ingress queue by acknowledging the message, the sender will eventually be unable to send more messages i.e. append to its egress queue, enforced by the relay-chain trusted logic.

Expected usage
--------------

Every sending parachain may send up to ~1 MB per chain height in total, to all parachains. In the most unbalanced case, this will be to a single recipient parachain.

TODO: in the worst case, (C-1) parachains will each send ~1 MB to the same victim receiver parachain. We could attempt to limit this.

TODO: chains can only communicate when they've opened a channel to each other, the state of which is stored on-chain. We can potentially use this information to derive more efficient topologies for XCMP.

Fairness
--------

A small digression: fairness means that receivers must process received messages fairly across all senders, and is mostly to ensure that no message will be left unprocessed for an infinite delay - the sender knows that the receiver must least ack its contents eventually, though they can drop the message after that. This is a value judgement made at the point-of-design of XCMP; we'll monitor its performance in practise.

Although different from the internet's recipient-controlled processing, fairness does not introduce much overhead since for global ordering and reliability, message-passing is co-ordinated via the relay chain anyways, and enforcing fairness on top of this is straightforward.

Requirements
============

R1. At least one message from every (non-empty) ingress queue must be transferred to the corresponding recipient, so that they may perform their obligation to ack at least one message.

R2. Ideally, allow recipients to select which message(s) to receive first, subject to the fairness constraints mentioned above.


Comparison with A&V
===================

Similarities
------------

Data flow pattern, i.e. outboxes to inboxes

Differences
-----------

Data usage profile

Less overall traffic, but much greater variability

Latency not such a big deal, can be similar to A&V, but in practise should
complete quicker due to less overall traffic.


More notes in https://hackmd.io/9JvbSmiNTiGUqEjQiPWtKQ?both
