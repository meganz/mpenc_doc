===============
Progress report
===============

Security
========

We do not try to provide confidentiality of session existence, membership, or
metadata, under any attack scenario. However, we try to avoid incompatiblities
with any future other systems that do provide these properties, based on our
knowledge of existing research and systems that approximate these.

We aim to directly provide confidentiality of message content and authenticity
of membership, ordering, and content; and later a limited form of confidential
authenticity of ordering and content. An overview follows; we will expand on it
in more detail in subsequent sections.

We achieve authenticity and confidentiality of message content with standard
modern cryptographic primitives using ephemeral session keys. The security of
these keys, and the authenticity of session membership and boundary ordering
(freshness), are achieved through a group key agreement and authentication
protocol. At present we use our own protocol, but this could be replaced.

The authenticity of message ordering is achieved through the authenticity of
message content, together with some rules that enforce logical consistency.
That is, someone who can authenticate message contents (either an attacker via
secrets leak or a corrupt insider) must still adhere to those rules.

The authenticity of message membership (reliability, consistency) is achieved
through the authenticity of message ordering, together with some rules that
enforce liveness, using timeout warnings and continual retries. This *differs*
from previous approaches (TODO: give examples) which try to achieve these
properties through the session establishment protocol; we believe that they are
more appropriately dealt with here, as will be justified later.

Additionally, our choice of mechanisms are intended to offer different levels
of protection for these properties, under various attack scenarios:

Under active communications attack:
  Retain all security properties mentioned above, for all members.

Under identity secrets leak against some targets (and active attack):
  Older sessions:
    - Retain all relevant security properties.

  Current sessions:
    - Retain all relevant security properties until the next membership change;
    - [+] From the next change onwards, properties are as per *newer sessions*,
      since in our system this requires identity secrets as well.

  Newer sessions:
    - [x] Attacker can open/join sessions as targets, read and participate;
    - Attacker *cannot* open/join sessions as non-targets, read or participate;
    - Retain all relevant security properties, for sessions whose establishment
      was not actively compromised.

Under session secrets leak against some targets (and active attack):
  Older sessions:
    - Retain all relevant security properties.

  Current sessions:
    - [+; partly x] Attacker can read session events;
    - [+; partly x] Attacker can participate as targets;
    - Attacker *cannot* participate as non-targets.

  Newer sessions:
    - [+] as per *identity secrets leak: new sessions*; our session secrets
      unfortunately include identity secrets.

Under insider corruption (and under active attack):
  As per *session secrets leak*; but entries marked imperfect cannot be
  improved upon. More specifically, the properties we *do* retain are:

  - Some limited protection against false claims/omissions about ordering;
  - (upcoming work) Retain confidential authenticity of ordering and content.

  Note that these apply to all lesser attacks too; we mention them explicitly
  here so that this section is less depressing.

| [x] unavoidable, as explained in the previous chapter.
| [+] imperfect, theoretically improvable, but we have no immediate plans to.

Distributed systems
===================

Security properties are meant to detect *incorrect behaviour*; but conflicts in
naively-implemented distributed systems can happen *even when* everyone behaves
correctly. If we don't explicitly identify and resolve these situations as *not
a security problem*, but instead allow our security mechanisms to warn or fail
in their presence, we reduce the usefulness of the latter. In other words, a
warning which is 95% likely to be a false positive, is useless information and
a very bad user experience that may push them towards less secure applications.

Generally in a distributed system, events may happen concurrently, so ideally a
partial order (directed acyclic graph) rather than a total order (line) should
be used to represent the ordering of events. We do this for messages, so this
component of our system may be re-used in asynchronous systems, without major
redesign.

However, group key agreements (which is how we implement membership changes)
have traditionally not been developed with this consideration in mind. The main
problem that needs to be solved here, would be to define a method to merge the
results of two concurrent operations, or even the operations themselves. We
have not tried to do this, as it seems complicated and highly dependent on the
GKA used. Instead, we have developed a system to enforce and verify that these
operations happen in a consistent total order, that has zero bandwidth cost and
generalises to *any* GKA (that satisfies certain constraints).

Beyond this, there are several more non-security distributed systems issues,
that relate to the integration of cryptographic logic in the context of a group
transport channel (commonly implemented by insecure group messaging systems),
that we must figure out graceful low-failure-rate solutions for:

- When a user enters the channel during a GKA, what should they do?
- When a user leaves the channel during a GKA, what do others do?
- A member might start a GKA and send the first packet to the other channel
  members M, but when others receive it perhaps the channel membership is now
  different from M, or there was another operation that jumped in "before" it.
  How should we detect and resolve this?
- What happens if different GKA packets (initial or final), are decodeable by
  different members? Some of them may not yet be part of the cryptographic
  session. If we're not careful, they will think different things about the
  state of the session, or of others' sessions.
- What if some of the above things happen at the same time?

Individual solutions to each of these are fairly straightforward, but making
sure that these interact with each other in a sane way is not so. Then, there
is the task of describing the intended behaviour *precisely*. Only when we have
a precise idea on what is *supposed* to happen, can we construct a concrete
system that isn't fragile, i.e. require mountains of patches for corner cases
ignored during the initial hasty naive implementations.

- TODO: Multi-device support... what issues here?

Design and mechanism
====================

A session is a local process from the point of view of *one member*. We don't
attempt to represent a single "global" or "group" view of a session; in general
such a concept is not useful for modelling security *or* distributed systems.

A session, time-wise, contains a linear sequence of *session membership change*
operations, that each take some interval of time to finish (and if/when having
done so sucessfully) each create subsessions of static membership where members
may send messages to each other perhaps even concurrently in a partial order.

Each operation G is initialised from (a) previous state (either a null state,
or the result of the previous operation) that encodes the current membership M'
and any cryptographic values that G needs such as ephemeral keys; and (b) the
first packet of the operation, sent by a member of M', which defines the
intended next membership M. When G is still ongoing, members may send messages
in the current subsession as normal, i.e. to M'. [#atom]_

G may finish with success, upon which we atomically change to a new subsession
for M, and the result state is stored for the next operation; or with failure,
upon which we destroy all temporary state related to G, and continue using the
existing subsession with membership M'.

After a successful change, the now-previous subsession (with membership M')
enters a shutdown phase. This happens concurrently and independently of other
parts of the session, such as messaging in the new subsession or subsequent
membership change operations on top of G.

The local process that runs a session consists of several internal components:

- a client interface to the group transport channel; this is the only interface
  the process has with the network;
- a component that manages membership operations as described above, storing
  state between operations and creating/destroying subprocesses to run them;
- two components for the current and previous subsession that process, store,
  and generate message packets;
- a concurrency resolver, to gracefully prevent conflicts caused by members
  trying to perform membership changes concurrently.

The process handles internally the various cases mentioned above relating to
transport integration, helped by the concurrency resolver. It also manages the
membership changes that are initiated by the local user, which require a bit
more hand-holding, such as retries in the case of transport hiccups, etc.

Each subsession component consists of:

- a message encryptor/decryptor for communicating to/from the session;
- a transcript data structure to hold accepted messages in the correct order;
- various liveness components to ensure end-to-end reliability and consistency.

The session receive handler roughly runs as follows. For each incoming packet:

1. if it represents a channel membership change, then react to it (i.e. as part
   of transport integration), which we'll go into in more detail later;
2. else, if it is a membership operation packet:

   - if it is relevant to the concurrency resolver, pass it to that, which may
     cause an operation to start or finish (with success or failure);
   - if an operation is ongoing, pass it to the subprocess running that;
   - else reject the packet - it's not appropriate at this time.

3. else, try to decrypt it as a message in the current subsession;
4. else, try to decrypt it as a message in the previous subsession;
5. else, put it on a queue, to try this process again later, in case it was
   received out-of-order and depends on missing packets to decrypt.

Out of these, the components that deal directly with cryptography are:

- the membership operations manager, that implements the group key agreement;
- the message encryptor/decryptor, that could implement a messaging ratchet.

These may be improved independently from the rest of the session components.
Furthermore, within each component, we may swap out cryptographic primitives -
i.e. DH key exchange schemes, signature schemes, hash functions and symmetric
ciphers - as necessary, based on the recommendations of the wider community.

.. [#atom] Our first group key agreement implementation did not enforce atomic
    operations. This caused major problems when users would leave the channel
    at different times, e.g. if they disconnect or restart the application,
    since their GKA components would see different packets and reach different
    states. With atomic operations and our transport integration rules, an
    inconsistent state is only reached if the transport (e.g. chat server)
    behaves incorrectly. That is, security warnings fire only when *there is
    actually a problem*, one of our goals.

Group key agreement
-------------------

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
---------------------

On top of the main membership change protocol, we have the initial packet of
each operation reference the final packet of the previous successful operation
(or null for the first one). The concurrency resolver then simply accepts the
earliest packet in the channel with a given parent reference, and rejects other
such packets. There are also more issues such as what to do when new members
enter the channel, since they might have missed previous such packets.

To solve the other distributed systems issues raised earlier, we have a system
of rules on how to react to different channel and session events. These work
roughly along these lines:

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
----------------

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

For a more detailed exploration, including tradeoffs of the "defer processing"
approach to strong ordering, and ways to calculate references to have better
resistance against false claims, see [TODO: link].

Reliability and consistency
---------------------------

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
------------------

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
