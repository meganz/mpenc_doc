========
Protocol
========

Here we present an overview of our protocol, suggested runtime data structures
and algorithms, with references to other external documents for more specific
information about each subtopic.

Session overview
================

A session is a local process from the point of view of *one member*. We don't
attempt to represent a single "group" view of a session; in general such views
are not good for *accurately* modelling security *or* distributed systems.

A session, time-wise, contains a linear sequence of *session membership change*
operations, that (if and when completed successfully) each create subsessions
of static membership, where members may send messages to each other.

Linearity is enforced through an acceptance mechanism, which is applied to the
packets where members try to start or finish an operation. The sequence is also
context-preserving, i.e. it is guaranteed that no other operations complete
*between* when a member proposes one, and it being accepted by the group.

Each accepted operation G is initialised from (a) previous state (the result of
the previous operation, or a null state if the first) that encodes the current
membership M' and any cryptographic values that G needs such as ephemeral keys;
and (b) the first packet of the operation, sent by a member of M', that defines
the intended next membership M. While G is ongoing, members may send messages
in the current subsession as normal, i.e. to M'. [#atom]_

G may finish with success, upon which we atomically change to a new subsession
for M, and the result state is stored for the next operation; or with failure,
upon which we destroy all temporary state related to G, and continue using the
existing subsession with membership M'.

After a successful change, the now-previous subsession (with membership M')
enters a shutdown phase. This happens concurrently and independently of other
parts of the session, such as messaging in the new subsession or subsequent
membership change operations on top of G.

.. [#atom] Our first group key agreement implementation did not enforce atomic
    operations. This caused major problems when users would leave the channel
    at different times, e.g. if they disconnect or restart the application,
    since their GKA components would see different packets and reach different
    states. With atomic operations and our transport integration rules, an
    inconsistent state is only reached if the transport (e.g. chat server)
    behaves incorrectly. That is, security warnings fire only when *there is
    actually a problem*, one of our goals.

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

For the overall protocol, the number of communication rounds is O(n) in the
number of members. The average size of GKA packets is also O(n). More modern
protocols have O(1) number of rounds but retain O(n) packet size. However, our
protocol is a simple "first approach" using elementary cryptography only, which
should be easier to understand and review.

There is potential to add a weak form of deniability later, where authenticity
of message contents are deniable, but authenticity of session participation is
not. This is essentially the group analogue of how deniability is achieved in
OTR, and has equivalent security. This is explored in more detail later. More
modern techniques make the key agreement itself deniable (via a zero knowledge
proof) but we're not expert enough in cryptography to do that here.

Transport integration
=====================

We have the initial packet of each operation, reference the final packet of the
previous finished operation (or null if the first). Likewise, the final packet
of each operation references (perhaps implicitly, if this is secure) a specific
initial packet. The concurrency resolver simply accepts the earliest packet in
the channel with a given parent reference, and rejects other such packets.

There are some more issues, both in the algorithm (e.g. for new members that
don't know "the previous operation", and to authenticate the parent references)
as well as in the implementation (e.g. for 1-packet operations that are both
"final" and "initial); more details are covered in [TODO: link].

The advantage of this *implicit* mechanism is that it has zero bandwidth cost,
generalises to *any* membership change protocol, and does not depend on the
liveness of a particular member as it would if we had an explicit leader.

To cover the other :ref:`distributed-systems` issues, we also have a system of
rules on how to react to various channel and session events. These work roughly
along these principles:

- Every operation's target members (i.e. new members and remaining members)
  must all be in the channel to see the first packet (otherwise it is ignored)
  and remain there for the duration of the operation (otherwise it auto-fails).

- Members that leave the channel are automatically excluded from the session,
  and vice versa. There are subrules to handle events that conflict with this
  auto-behaviour, that might occur before those behaviours are applied.

- We never initiate membership operations to exclude ourselves. When we want to
  part the session, we initiate a "shutdown" process on the subsession, wait
  for it to finish, then leave the channel. When others exclude us, we wait for
  them to kick us from the channel, if and after the operation succeeds. Either
  way, we switch to a "null/solo" subsession only *after* leaving the channel.

For full details of these rules and the rationale for them, along with more
precise descriptions of the model for a general G, and of the group transport
channel, see [TODO: link].

Message ordering
================

As discussed :ref:`previously <distributed-systems>` and elsewhere [TODO: link]
messages may be sent concurrently even in a group transport channel, and
therefore we represent the transcript of messages as a causal order.

Every message has an explicit set of references to the latest other messages
("parents") seen by the author when they wrote the message. These references
are hashes of each packet, which require no extra infrastructure to generate or
resolve. When we decrypt and verify a packet, we verify the author of these
references as well. This allows us to ignore the order of packet receipt, and
instead construct our ordering by following these references. If we receive a
packet out-of-order, i.e. if we haven't yet received all of its parents, we
simply defer processing of it until we have received them.

To be precise, it is important to note that parent references are only claims.
Their truth is susceptible to lying; the claimant may:

- make false claims, i.e. refer to messages they haven't seen; hashes give some
  protection here, but they could e.g. reuse a hash they saw from someone else;
- make false omissions, i.e. not refer to messages that they have seen.

We have rules that enforce some logical consistency here:

- a message's parents must form an anti-chain, i.e. none of these parents may
  directly or indirectly (via intermediate messages) reference each other;
- an author's own messages must form a total order (line).

This gives some protection against arbitrary lies, but it is still possible to
lie within these constraints. However, we don't offer protection for this; we
believe that there is no benefit for an attacker to make such lies, and that
the cost of any solution would not be worth the minor extra protection.

References must also have the property that the same reference to a packet is
interpreted (decrypted and verified into content, parents and membership) by
all of its members identically. Our simple hash-based definition, together with
using a shared group encryption key, guarantees this for our case.

For a more detailed exploration, including tradeoffs of the "defer processing"
approach to strong ordering, and ways to calculate references to have better
resistance against false claims, see [TODO: link].

Reliability and consistency
===========================

Due to our strong ordering property, we can interpret parent references as an
implicit acknowledgement ("ack") that the author received every parent. Based
on this, we can ensure end-to-end reliability and consistency; we take much
inspiration from the core ideas of TCP.

We require every message (those we send, *and* those we receive) to be acked by
all recipients; if we (as the local user) don't observe these within a timeout,
we warn the human user. We may also occasionally resend the packets of these
messages, possibly including others' packets that we received.

To ensure that we ack everything that everyone sent, we also occassionally send
out acks automatically outside of the user's control. Due to strong ordering,
acks are transitive (i.e. implicitly ack all of its ancestors) and thus these
auto-acks can be delayed, to ack several messages at once and reduce volume.

There are more considerations, to avoid perpetual reacking-of-acks but ensure
that the final messages of a session, or of a busy period within a session, are
actually fully-acked. This includes a formal session "shutdown" process.

For a more detailed exploration, including resend algorithms, timing concepts,
different ack semantics, why we must have end-to-end authenticated reliability,
and the distinction between consistency and consensus, see [TODO: link].

Message encryption
==================

Message encryption is currently very simple. Each subsession has a constant set
of keys (the output of the group key exchange) that are used to authenticate
and encrypt all messages in it - one encryption key shared between all members,
and one signature key for each member, with the public part shared with others.

Every message is encrypted using the shared encryption key, then signed by the
author using their own private signature key. To decrypt, the recipient first
verifies the signature, then decrypts the ciphertext.

These are constant throughout the session, so that if the shared encryption key
is broken, the confidentiality of message content is lost. In the future, we
will experiment with implementing this component as a forward secrecy ratchet.
Note that we already have forward secrecy *between* subsessions.

One simple scheme is to deterministically split the key into n keys, one for
each sender. Then, each key can be used within a hash-chain ratchet for the
corresponding sender. Once all recipients have decrypted a message and deleted
the key, the secrecy of messages encrypted with that key and previous ones is
ensured, even if an attack compromises members' memory later. However, since
this scheme does not distribute entropy between members, there is no chance to
recover from a memory leak and try to regain secrecy for future messages.

There is also the future option to make the message authentication confidential
("deniable"). Roughly speaking, once a member initiates a subsession shutdown
request ("FIN"), they may publish their signature key after everyone acks this
request. This is safe (an attacker cannot re-use the key to forge messages) if
we enforce that one may not author messages *after* a FIN, i.e. all receivers
must refuse to accept such messages. However, this simple approach destroys our
ability to authenticate our own acks of others' messages (e.g. *their* FIN)
after we send our own FIN. So we probably need something a bit more complex,
and we haven't worked out the details yet.

There is the attack here that if others' acks to our FIN are blocked, then we
will never be sure that it's safe to publish our signature key. This likely
can't be defended under this type of scheme, since confidential authenticity
isn't meaningful without authenticity (it would be "confidential nothing"); the
equivalent attack also applies to OTR. To defend against this, we would need a
session establishment protocol that is itself deniable, and then we don't need
to mess around with publishing the keys used for message authentication.
