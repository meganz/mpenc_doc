========
Protocol
========

Here we present an overview of our protocol, suggested runtime data structures
and algorithms, with references to other external documents for more specific
information about each subtopic.

Session overview
================

A session is a local process from the point of view of *one member*. We don't
attempt to reason about nor represent an overall "group" view of a session; in
general such views are not useful for *accurately* modelling, or implementing,
security or distributed systems.

A session, time-wise, contains a linear sequence of *session membership change*
operations, that (if and when completed successfully) each create subsessions
of static membership, where members may send messages to each other.

Linearity is enforced through an acceptance mechanism, which is applied to the
packets where members try to start or finish an operation. The sequence is also
context-preserving, i.e. it is guaranteed that no other operations complete
*between* when a member proposes one, and it being accepted by the group.

Each accepted operation :math:`G` is initialised from (a) previous state (the
result of the previous operation, or a null state if the first) that encodes
the current membership :math:`M'` and any cryptographic values that :math:`G`
needs such as ephemeral keys; and (b) the first packet of the operation, sent
by a member of :math:`M'`, that defines the intended next membership :math:`M`.
While :math:`G` is ongoing, members may send messages in the current subsession
as normal, i.e. to :math:`M'`. [#atom]_

:math:`G` may finish with success, upon which we atomically change to a new
subsession for :math:`M`, and store the result state for the next operation; or
with failure, upon which we destroy all temporary state related to :math:`G`,
and continue using the existing subsession with membership :math:`M'`.

After a successful change, the now-previous subsession (with membership
:math:`M'`) enters a shutdown phase. This happens concurrently and
independently of other parts of the session, such as messaging in the new
subsession or subsequent membership change operations on top of :math:`M`.

.. [#atom] Our first group key agreement implementation did not enforce atomic
    operations. This caused major problems when users would leave the channel
    at different times, e.g. if they disconnect or restart the application,
    since their GKA components would see different packets and reach different
    states. With atomic operations and our transport integration rules, an
    inconsistent state is only reached if the transport (e.g. chat server)
    behaves incorrectly. One of our goals is, that security warnings fire
    *only* when there is *actually* (or *likely* to be) a problem.

Group key agreement
===================

Our group key agreement is composed of the following two subprotocols:

- a group DH exchange to generate a shared ephemeral secret encryption key, for
  confidentiality;
- a custom group key distribution protocol for per-member ephemeral public
  signature keys, for authenticity (of session membership) and freshness,
  piggy-backed onto the same packets as the group DH packets.

The keys are used to read/write messages in the subsession created by the
operation. Identity secrets are *not* needed for this, but they are needed for
participating in further membership changes (i.e. creating new subsessions).

For the overall protocol, the number of communication rounds is :math:`O(n)` in
the number of members. The average size of GKA packets is also :math:`O(n)`.
More modern protocols have :math:`O(1)` rounds but retain :math:`O(n)` packet
size. However, our protocol is a simple first approach using elementary
cryptography only, which should be easier to understand and review.

This component may be upgraded or replaced independently of the other parts of
our protocol system. For example, more modern techniques make the key agreement
itself deniable via a zero-knowledge proof. We have skipped that for now to
solve higher-priority concerns first, and because implementing such protocols
correctly is more tricky. This is discussed further in :doc:`future-work`.

.. _transport-integration:

Transport integration
=====================

We have the initial packet of each operation, reference the final packet of the
previous finished operation (or null if the first). Likewise, the final packet
of each operation references (perhaps implicitly, if this is secure) a specific
initial packet. The concurrency resolver simply accepts the earliest packet in
the channel with a given parent reference, and rejects other such packets. We
also define a *chain hash* built from the sequence of accepted packets, that
members verify the consistency of after each operation finishes.

The advantage of this *implicit* agreement is that it has zero bandwidth cost,
generalises to *any* membership change protocol, and does not depend on the
liveness of a particular member as it would if we had an explicit leader.

There are further details and cases in both the algorithm and implementation.
For example, we have additional logic to handle new members who don't know the
previous operation, and define a general method to authenticate the parent
references. We also arrange the code carefully to behave correctly for 1-packet
operations, where the packet acts both as an initial and a final packet in our
definitions above.

To cover the other :ref:`distributed-systems` cases, we also have a system of
rules on how to react to various channel and session events. These work roughly
along these principles:

- Every operation's target members (i.e. joining members and remaining members)
  must all be in the channel to see the first packet (otherwise it is ignored)
  and remain there for the duration of the operation (otherwise it auto-fails).

- Members that leave the channel are automatically excluded from the session,
  and vice versa. There are subrules to handle events that conflict with this
  auto-behaviour, that might occur before those behaviours are applied.

- We never initiate membership operations to exclude ourselves. When we want to
  part the session, we initiate a shutdown process on the subsession, wait for
  it to finish, then leave the channel. When others exclude us, we wait for
  them to kick us from the channel, if and after the operation succeeds. Either
  way, we switch to a "null/solo" subsession only *after* leaving the channel.

For full details of our agreement mechanism, our transport integration rules,
and the rationale for them, along with more precise descriptions of our model
for a general :math:`G` and of a group transport channel, see [msa-5h0]_.

Message ordering
================

As discussed :ref:`previously <distributed-systems>` and elsewhere [msg-2i0]_
messages may be sent concurrently even in a group transport channel, and so we
represent the transcript of messages as a cryptographic causal order. We take
much inspiration from Git_ and OldBlue [12OBLU]_ for our ideas.

Every message has an explicit set of references to the latest other messages
("parents") seen by the author when they wrote the message. These references
are hashes of each packet, which require no extra infrastructure to generate or
resolve. When we decrypt and verify a packet, we verify the author of these
references as well. This allows us to ignore the order of packet receipt, and
instead construct our ordering by following these references. If we receive a
packet out-of-order, i.e. if we haven't yet received all of its parents, we
simply defer processing of it until we have received them.

References must at least be second-preimage-resistant, with the pre-image being
some function of the *full* verified-decrypted referent message (i.e. content,
parents, author and readers), so that all members interpret them consistently.

Our definition based on hashing packet ciphertext, together with using a shared
group encryption key, guarantees the above property for our case. However to be
precise, it is important to note that such references are only claims. Their
truth is susceptible to lying; the claimant may:

- make false claims, i.e. refer to messages they haven't seen; second pre-image
  resistance gives *some* protection here, but an attacker could e.g. reuse a
  hash value that they saw from another member;
- make false omissions, i.e. not refer to messages that they have seen.

We have rules that enforce some logical consistency across references:

- a message's parents must form an anti-chain, i.e. none of these parents may
  directly or indirectly (via intermediate messages) reference each other;
- an author's own messages must form a total order (line).

This gives some protection against arbitrary lies, but it is still possible to
lie within these constraints. Nevertheless, we omit protection for the latter,
since we believe that there is no benefit for an attacker to make such lies,
and that the cost of any solution would not be worth the minor extra security.

For a more detailed exploration, including tradeoffs of the *defer processing*
approach to strong ordering, and ways to calculate references to have better
resistance against false claims, see [msg-2o0]_.

.. _Git: https://git-scm.com/

Reliability and consistency
===========================

Due to our strong ordering property, we can interpret parent references as an
implicit acknowledgement ("ack") that the author received every parent. Based
on this, we can ensure end-to-end reliability and consistency. We take much
inspiration from the core ideas of TCP_.

We require every message (those we write, *and* those we read) to be acked by
all (other) readers; if we don't observe these within a timeout, we warn the
user. We may occasionally resend the packets of those messages (the subjects of
such warnings), including those authored by others. Resends are all based on
implicit conditions; we have no explicit resend requests as in OldBlue.

To ensure we ack everything that everyone wrote, we also occassionally send
acks automatically outside of the user's control. Due to strong ordering, acks
are transitive (i.e. implicitly ack all of its ancestors) and thus auto-acks
can be delayed to "batch" ack several messages at once and reduce volume.

We develop some extra details to avoid perpetual reacking-of-acks, yet ensure
that the final messages of a session, or of a busy period within a session, are
actually fully-acked. We also include a formal session shutdown process.

For a more detailed exploration, including resend algorithms, timing concepts,
different ack semantics, why we must have end-to-end authenticated reliability
instead of "just using TCP", the distinction between consistency and consensus,
and more, see [msg-2c0]_.

.. _TCP: https://en.wikipedia.org/wiki/Transmission_Control_Protocol

Message encryption
==================

For now, message encryption is very simple. Each subsession has a constant set
of keys (the output of the group key exchange) that are used to authenticate
and encrypt all messages in it -- one encryption key shared across all members,
and one signature key for each member, with the public part shared with others.

Every message is encrypted using the shared encryption key, then signed by the
author using their own private signature key. To decrypt, the recipient first
verifies the signature, then decrypts the ciphertext.

These are constant throughout the session, so that if the shared encryption key
is broken, the confidentiality of message content is lost. In the future, we
will experiment with implementing this component as a forward secrecy ratchet.
Note that we already have forward secrecy *between* subsessions.

There is also the option to add a weak form of deniability, where authenticity
of message contents are deniable, but authenticity of session participation is
not. This is essentially the group analogue of how deniability is achieved in
OTR [OTR-spec]_, and has equivalent security. (As mentioned before, making the
group key agreement itself deniable is stronger, but more complex to achieve.)
These directions are discussed further in :doc:`future-work`.
