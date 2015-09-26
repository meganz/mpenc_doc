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
of membership, ordering, and content; as well as a limited form of confidential
authenticity of ordering and content, explained in more detail below.

Specifically, we achieve authenticity and confidentiality of message content
via standard modern cryptographic primitives. The security of these, as well as
the authenticity of session membership and boundary ordering (freshness), are
achieved through a group key agreement and authentication protocol. Currently
we use our own protocol, but others could be swapped in at a future date.

The authenticity of message ordering is achieved through the authenticity of
message content, together with some rules that enforce logical consistency.
That is, someone who can authenticate message contents (either an outside
attacker via session secrets leak or a corrupt insider) must still adhere to
the latter rules. This is explained in more detail later.

The authenticity of message membership (reliability, consistency) is achieved
through the authenticity of message ordering, together with some rules that
enforce liveness, using timeout warnings and continual retries. This *differs*
from previous approaches (TODO: give examples) which try to achieve these
properties through the session establishment protocol; we believe that they are
more appropriately dealt with here, as explained elsewhere.

Additionally, our choice of mechanisms are intended to offer different levels
of protection for these properties, under various attack scenarios:

Under active communications attack:
  Retain all security properties mentioned above, for all members.

Under identity secrets leak against some targets (and active attack):
  Older sessions:
    - Retain all relevant security properties.

  Current sessions:
    - Retain all relevant security properties until the next membership change.
    - From the next membership change onwards, as per "newer sessions", since
      in our system this requires identity secrets as well.

  Newer sessions:
    - (not achieved; unavoidable) Attacker can open/join sessions as targets
    - Attacker *cannot* open/join sessions as non-targets.
    - Retain all relevant security properties, for sessions whose establishment
      was not actively compromised.

Under session secrets leak against some targets (and active attack):
  Older sessions:
    - Retain all relevant security properties.

  Current sessions:
    - (not achieved; partly unavoidable) Attacker can read session events;
    - (not achieved; partly unavoidable) Attacker can participate as targets;
    - Attacker *cannot* participate as non-targets.

  Newer sessions:
    - as per "identity secrets leak: new sessions"; our session secrets
      unfortunately include identity secrets.

With a corrupt insider (and under active attack):
  - (future work) Retain confidential authenticity of ordering and content;
  - Some limited protection against false claims/omissions about ordering;
  - Otherwise, as per "session secrets leak".

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
component of our system may be re-used in asynchronous scenarios, without a
major redesign.

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
transport channel (as commonly implemented by non-end-to-end secure messaging
systems), that we must figure out graceful low-failure-rate solutions for:

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

- Multi-device support... what issues here?

Design and mechanism
====================

A session is a local process from the point of view of *one member*; we have
nothing to present a "global" or "group" view of a session, and such a concept
is not useful for modelling security *or* distributed systems in general.

A session, time-wise, contains a linear sequence of "session membership change"
operations, that each spawn sub-sessions of static membership where members may
send messages to each other, perhaps even concurrently in a partial order.

Each operation G is initialised from some previous state (either a null state,
or the result of the previous operation) that encodes the current membership M'
and any cryptographic values that G needs such as ephemeral keys; as well as
the first packet of the operation, sent by a member of M', which defines the
intended next membership M. When G is still ongoing, members may send messages
in the current subsession as normal, i.e. to M'.

G may finish with success, upon which we atomically change to a new subsession
for M, and the result state is stored for the next operation; or with failure,
upon which we remain at M' and destroy all temporary state related to G.

After a successful change, the now-previous subsession (with membership M')
enters a shutdown phase. This happens concurrently and independently of other
parts of the session, such as messaging in the new subsession or subsequent
membership change operations.

The local process that runs a session consists of several internal components:

- a client interface to the group transport channel; this is the only interface
  the process has with the network;
- a component that manages membership operations as described above, storing
  state between operations and creating/destroying subprocesses to run them;
- two components for the current and previous subsession that process, store,
  and generate messages;
- a concurrency resolver, to gracefully prevent conflicts caused by members
  trying to perform membership changes concurrently.

The process handles internally the various cases mentioned above relating to
transport integration, helped by the concurrency resolver. It also manages the
membership changes that are initiated by the local user, which require a bit
more hand-holding, such as retries in the case of transport hiccups, etc.

Each subsession component consists of:

- a message encryptor/decryptor; currently this only runs simple operations
  using keys constant over the subsession, but in the future could be a
  messaging ratchet that provides inter-subsession forward secrecy;
- a transcript data structure, to hold accepted messages in the correct order;
- various liveness components to ensure end-to-end reliability and consistency.

The session receive handler roughly runs as follows. For each incoming packet:

1. try to decrypt it as a message in the current subsession; otherwise
2. try to decrypt it as a message in the previous subsession; otherwise
3. if it is a membership operation packet:

   - if it is relevant to the concurrency resolver, pass it to that, which may
     cause an operation to start or finish (with success or failure)
   - if an operation is ongoing, pass it to the subprocess running that

4. if it represents a channel membership change, then react to it (i.e. as part
   of transport integration), which we'll go into in more detail later.

In summary, the components that deal directly with cryptography are:

- the message encryptor/decryptor
- the membership operation manager

These may be improved independently from the rest of the session components.
Furthermore, within each component, we may swap out cryptographic primitives as
necessary, based on the recommendations of the wider community - namely DH
exchange keys, signature keys, hash functions and symmetric ciphers.

Group key agreement
-------------------

GKA to generate group enc key / eph sig keys (integrate previous paper).

Long-term keys not necessary for messages, but necessary for membership changes.

Transport integration
---------------------

Extra system to prevent races (ServerOrder, link to msg-notes)

Message ordering
----------------

Parent pointers

It is still possible to lie within the constraints of these rules:

- make false non-receipt claims (not referring to seen messages)
- make false receipt claims (referring to unseen messages)

But we don't offer more protection based on a cost benefit trade off - we
believe that there is no benefit to be gained by an attack here. There is some
discussion on how to prevent this, though [link to msg-notes], which is quite
costly, but perhaps someone could explore it in the future. TODO: rw prev

Reliability and consistency
---------------------------

Timeout warnings for "liveness" properties

Message encryption
------------------

Message encryption, deniability, publishing signature keys.
