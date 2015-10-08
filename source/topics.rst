======
Topics
======

This chapter describes the goals for our messaging protocol, the constraints we
chose, and a discussion of the issues that occur under these contexts.

Security goals
==============

We do not try to provide confidentiality of session existence, membership, or
metadata, under any attack scenario. However, we try to avoid incompatiblities
with any future other systems that do provide these properties, based on our
knowledge of existing research and systems that approximate these. (We do use
an opportunistic exponential padding scheme that hides the lowest bits of our
content length, but have not analysed this under a strong threat model.)

We aim to directly provide confidentiality of message content and authenticity
of membership, ordering, and content; and later a limited form of confidential
authenticity of ordering and content. An overview follows; we will expand on it
in more detail in subsequent chapters.

We achieve authenticity and confidentiality of message content, by using modern
standard cryptographic primitives with ephemeral session keys. The security of
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
from previous approaches [#gotr]_ which try to achieve these properties through
the session establishment protocol; we believe that they are more appropriately
dealt with here, as will be justified later.

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

.. [#gotr] TODO: give links and examples

.. _distributed-systems:

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
causal order (directed acyclic graph) rather than a total order (line) should
be used to represent the ordering of events. We do this for messages, so this
component of our system may be re-used in asynchronous systems, without major
redesign.

However, group key agreements (which is how we implement membership changes)
historically have not been developed with this consideration in mind. An ideal
solution would define how to merge the results of two concurrent operations, or
even how to merge the operations themselves. But we could not find literature
that even mentions this problem, and we are unsure how to begin to approach it.
Based on our limited knowledge in this area, it seems reasonable at least that
any solution would be highly specific to the GKA used, limiting our future
options for replacing cryptographic components.

For now, we give up causal ordering for membership operations, and instead
enforce a linear ordering for them. To make this easier, we restrict ourselves
to a group transport channel, that takes responsibility for the delivery of
channel events (messages and membership changes), reliably and in a consistent
order. [#xmpp]_ We do not *assume* this; we detect any deviation from it and
notify the user, but our system is efficient if the transport is honest.

Beyond this, there are several more non-security distributed systems issues,
that relate to the integration of cryptographic logic in the context of a group
transport channel. These situations all have the potential to mess up the state
of a naive implementation:

- Two members start different operations concurrently (as mentioned above);
- Two members try to complete an operation concurrently, in different ways;
  e.g. "send the final packet" vs "send a request to abort";
- A user enters the channel during an operation;
- A user leaves the channel during an operation;
- A member starts an operation and sends the first packet to the other channel
  members M, but when others receive it the membership has changed to M', or
  there was another operation that jumped in before it;
- Different operation packets (initial or final) are decodeable by different
  members, some of which are not part of the cryptographic session. If we're
  not careful, they will think different things about the state of the session,
  or of others' sessions;
- Any of the above things could happen at the same time.

We must design graceful low-failure-rate solutions for all of them. Individual
solutions to each of these are fairly straightforward, but making sure that
these interact with each other in a sane way is more complex. Then, there is
the task of describing the intended behaviour *precisely*. Only when we have a
precise idea on what is *supposed* to happen, can we construct a concrete
system that isn't fragile, i.e. require mountains of patches for corner cases
ignored during the initial hasty naive implementations.

.. [#xmpp] For example, XMPP MUC would be suitable for this purpose, since one
    single server keeps a consistent order for the channel. In IRC, there may
    be multiple servers that opportunistically bounce messages from clients
    without trying to agree on a consistent order.

User experience
===============

Independently of any actual attack or security warning, the distributed nature
of our system requires us to consider how to represent *correct* information to
users. Displaying inaccurate or vague information is a security risk *even
without an attacker* because it can lead the user to believe incorrect things.

Here, we give an overview of these issues and our suggested solutions for them.
Due to time constraints, we have not yet implemented these; but none of the
options seem hard to construct, or complex for user experience. Avoiding any of
these topics is always an option, which case the application will look like
*and be as insecure as*, existing applications that do the same.

A more detailed discussion is given at [TODO: link].

Real parents of a message
  Some messages may not be displayed immediately below the one(s) that they are
  actually sent after, i.e. that the author saw when sending it.

  Our suggestion: (a) allow the user to select a message (e.g. via mouse click,
  long press or keyboard) upon which all non-ancestors are grayed out; and (b)
  annotate the messages whose parents are not equal to the set { the preceding
  message in the UI }, as a hint for the user to perform the selection.

Messages sent before a membership change completes, but received afterwards
  Obviously, this message has a different membership from the current session,
  and it would be wrong not to display this difference.

  Our suggestion: (a) when an operation completes, issue a UI notice about it
  inline in the messages view; (b) allow the user to select a message to see
  its membership, instead of trying to infer it from the session membership and
  any "change" notices; and (c) annotate such messages as a hint for the user
  to perform the selection.

Progress and result of a membership change operation
  If the user starts an operation then immediately sends a message, this is
  still encrypted to the *old* membership. Unless we explicitly make it clear
  that operations take a finite time, they may not realise this.

  Our suggestion: issue UI notices inline in the messages view, when the user
  proposes an operation and when it is rejected, is accepted (starts), fails or
  succeeds; or (optionally) also when *others'* operations are rejected, are
  accepted, fail or succeed.

Messages received out-of-order
  Some messages are sent, but the sent-later ones are received earlier.

  Our suggestion: simply ignore the messages that are received too early, until
  the missing gaps are filled. This might seem counter-intuitive, but there are
  many reasons that this is the best behaviour, discussed [TODO: link]. There
  are some other options, but we believe these are all strictly worse.

Messages not yet acknowledged by all of its intended recipients
  Here, we are unsure if everyone received what we sent, or received the same
  messages that we received from others.

  Our suggestion: (a) allow the user to select a message to see who has not yet
  acknowledged it, out of its membership; (b) annotate such messages as a hint
  for the user to perform the selection, after a grace timeout because it's
  impossible to satisfy this immediately; and optionally (c) show a progress
  meter for this condition for every message we send.

Users not responding to heartbeats
  This helps to detect transports dropping our messages.

  Our suggestion: in the users view, gray out expired users.
