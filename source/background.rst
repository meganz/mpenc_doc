.. include:: <mmlalias.txt>

==========
Background
==========

This chapter is not about our work, but a general discussion of secure group
communications -- our model of what it is, and "ideal" properties that *might*
be achieved. It is the longest chapter, so readers already familiar with such
topics may prefer to skip to the next one.

Often, lists of security properties can seem arbitrary, with technical names
that seem unrelated to each other. We take a more methodical approach, and try
to classify these properties within a general framework. To be clear, this is
neither formal nor precise, and our own project goals only focus on a subset of
these. Our motivation is to *enumerate* all options from a *protocol design*
perspective, for future reference and for comparison with other projects that
focus on a different subset. It offers some assurance that we haven't missed
anything, provides better understanding of relationships between properties,
and suggests natural separations for solving different concerns.

There is other work, previous and ongoing, that gives more precise treatments
of the topics below. We encourage interested readers to explore those for
themselves, as well as future research on classification and enumeration.

Model and mechanics
===================

First, we present an abstract conceptual model of a *private group session* and
introduce some terminology. Secure communication systems generally consist of
the following steps:

0. Identity (long-term) key validation.
1. Session membership change (e.g. establishment or optional termination).
2. Session communication.

We only concern ourselves with (1) and (2). We assume that (0), a.k.a. the "PKI
problem", is handled by an external component that the application interacts
directly with, bypassing our components. Even if it is not handled safely, this
does not affect the *functionality* of our components; but the application
should display a warning [#keyv]_ and record this fact and/or have the user
complete that step retroactively.

A **session** (as viewed by a subject member, at a given time) is formed from a
set of transport *packets*, which the protocol interprets logically as a set of
messages and a *session membership*. Each **message** has a single *author* and
a set of *readers*; the *message membership* is the union of these. Messages
have an order relative to each other, that may be represented (without loss of
information) as a set of *parent messages* for each message; and relative to
the session boundary, i.e. when the subject decides to join and part.

Whilst part of a session, members may change the session membership, send new
messages to the current session membership, and receive these events from each
other. Events are only readable to those who are part of the session membership
of the event generator, when they generated it. For example, joining members
cannot read messages that were written before the author saw them join (unless
an old member leaks the contents to them, outside of the protocol). [#rejn]_

On the transport level, we assume an efficient packet broadcast operation, that
costs time near-constant in the number of recipients, and bandwidth near-linear
(or less) in the number of recipients or the size of the packet. Note that the
*sender* and *recipients* of a transport packet are concepts distinct from the
author and readers of a logical message; our choice of terminology tries to
make this clear and unambiguous.

For completeness, we observe that a real system often unintentionally generates
"side channel" information, beyond the purpose of the model. This includes the
time, space, energy cost of computation; the size, place of storage; the time,
size, route of communications; and probably more that we've missed. Often it is
impossible to avoid generating some of this information.

In summary, we've enumerated the broad categories of information in our model:
membership, ordering, content, and side channels. Next, we'll discuss and
classify the security properties we might want, then consider these properties
in the context of each of these categories.

.. [#keyv] For example, "WARNING: the authenticity and privacy of this session
    is dependant on the unknown validity of the binding $key |LeftRightArrow|
    $user" or perhaps something less technical.

.. [#rejn] We don't yet have a good model of what it should precisely mean to
    *rejoin* a session. This "happens to work" with what we've implemented, but
    is not easily extensible to asynchronous messaging. Specifically, it's
    unclear how best to consistently define the relative ordering of messages
    across all members. We will explore this topic in more depth in the future.

Security properties
===================

In any information network, we produce and consume information. This could be
explicit (e.g. contents) or implicit (e.g. metadata). From this very general
observation, we can suggest a few fundamental security properties:

**Efficiency**
  Nobody should be able to cause anyone to spend resources (i.e. time, memory,
  or bandwidth) much greater than what the initiator spent.

**Authenticity**
  Information should be associated with a proof, either explicit or implicit,
  that consumers may use to verify the truth of it.

**Confidentiality**
  Only the intended consumers should be able to access and interpret the
  information.

We'll consider authenticity and confidentiality as applied to the categories of
information we listed above. (Efficiency is more fiddly and we'll discuss it in
narrower terms later, applied to specific parts of our protocol system.)

Here is a summary of current known best techniques for achieving each property.
Though we don't try to achieve all of them in our protocol system, being aware
of them allows us to avoid decisions that destroy the possibility to achieve
them in the future.

+-------------------+-------------------+-------------------+
|                   | Authenticity      | Confidentiality   |
+===================+===================+===================+
| Existence         | automatic         | research topic    |
+-------------------+-------------------+-------------------+
| Side channels     | unneeded          | research topic    |
+-------------------+-------------------+-------------------+
| Membership        | via crypto        | research topic    |
+-------------------+-------------------+-------------------+
| Ordering          | via crypto        | not explored yet  |
+-------------------+-------------------+-------------------+
| Contents          | via crypto        | via crypto        |
+-------------------+-------------------+-------------------+

Auth. of session existence
  This is achieved automatically by authenticity of any of the other types; we
  don't need to worry about it on its own.

Conf. of session existence
  This is the hardest to achieve, and is an ongoing research topic. This is the
  scenario where the user must hide the fact that they are merely *using the
  protocol*, even if the attacker knows nothing about any actual sessions. Not
  only does it require confidentiality of all the other types of information,
  but also obfuscation, steganography, and/or anti-forensics techniques.

Auth. of side channels
  We don't care about the authenticity of something we didn't intend to
  communicate in the first place.

Conf. of side channels
  Still a research topic, this is a concern because it may be used to break
  the confidentiality of other types of information.

Auth. of membership
  There is some depth to this. The first choice is whether a distinct *change
  session membership* operation should exist outside of sending messages. "Yes"
  means that (e.g.) you can add someone to the session, and they will know this
  (maybe a window will pop up on their side) even if you don't send them any
  messages. "No" means that membership changes must always be associated with
  an actual message that effects this change. This is up to the application;
  though "no" is generally more suited for asynchronous messaging.

  If "yes", we must consider *entity aliveness* in our membership protocol.
  This is the property that *if* we complete the protocol successfully, *then*
  we are also sure that our peers have done the same thing. This requires a key
  confirmation step from joining members, making the protocol last at least 2
  rounds. If we don't consider this, then we may use shorter protocols, but
  then our peers might not know that we changed the session membership until we
  send them a message, which makes the "yes" choice less useful.

  In both cases, authenticating (adding a proof of authorship to) messages is
  not enough to verify membership, since packets may get dropped, perhaps even
  against different recipients. We need to wait for all readers to send a reply
  back to acknowledge their receipt. This is known as *reliability* (when the
  author checks it) or *consistency* (when a reader checks the other readers).

Conf. of membership
  This is more commonly called *unlinkability* and is an ongoing research
  topic. One major difficulty is that the information may be inferred from many
  different sources, often implicit in the implementation or in the choice of
  transport, and not explicit in the model. For example:

  - It may be inferred from side channels, such as timing or packet size
    correlations at the senders and recipients, or transport-defined headers
    that contain routing information. So, we need routing anonymity and padding
    or chaff mechanisms to protect against this line of attack.

  - It may be inferred from content, such as raw signatures or public keys.
    So, we need confidential authentication mechanisms (defined later) as well
    as obfuscated transport protocols to protect against this line of attack.

Auth. of ordering
  The two types of ordering we identified earlier are: ordering of messages
  within a session, and session boundary ordering relative to local events. The
  latter is more commonly known as *freshness*.

  Other systems sometimes claim freshness based on some idea of an absolute
  clock, but this requires trusting third-party infrastructure and/or the
  user's local clock being correct. We prefer to avoid such approaches as it
  makes the guarantees less clear.

  Cryptographic guarantees are best; e.g., if we witness (in *any order*) a
  hash and its pre-image, then we are sure that the former was generated (i.e.
  authored) *after* the latter. To link remote events to our own time, we can
  arrange for the pre-image to be derived from some unpredictable local event
  such as generating a random nonce.

Conf. of ordering
  This hasn't been explored explicitly in the wider literature, but could help
  to break confidentiality of other types of information. It's out of scope for
  us to consider it in more detail right now; sorry.

Auth. of contents
  This is a straightforward application of cryptography. To be clear, this is
  about proving that the author intended to send us the contents. Whether the
  claims in it are true (including any implicit claim that the contents weren't
  copied from elsewhere) is another matter. We'll refer back to this later.

Conf. of contents
  This is a straightforward application of cryptography.

In conclusion, we've taken a top-down approach to identify security properties
that are direct high-level user concerns in a general *private group session*.
We have not considered lower-level properties (e.g. *contributiveness*, *key
control*) here since they are only relevant to specific implementations; but we
will discuss such properties *when and if* they might affect the ones above.

Threat models
=============

Attackers with different powers may try to break any of the above properties.
First, let's define and describe these powers:

Active communications attack (on the entire transport)
  This is the standard attack that all modern communications systems should
  protect against -- i.e. a transport-level "man in the middle" who can inject,
  drop, replay and reorder packets. Generally (and in practise mostly) channels
  are bi-directional -- so we must assume that the attacker, if they are able
  to target one member, then they are able to target all members, by attacking
  the channel in the opposite direction.

  Since this a baseline requirement, the powers defined below should be taken
  to *include* the ability to actively attack the transport during any attack.

Leak identity secrets (of some targets)
  This refers to all of the secret material needed for a subject to establish a
  new session, e.g. with someone that they've never communicated with before.
  By definition, this is secret only to the subject.

Leak session secrets (of some targets)
  This refers to all of the secret material needed for a subject to continue
  participating in a session they're already part of. This may be secret to
  only the subject, or shared across all members, or a combination of both.

  This may *include* identity secrets if they must be used to generate/process
  messages or membership changes. Better protocols would *not* need them, so
  that they may be wiped from memory during a session, reducing the attack
  surface. However, even in the latter case, session secrets still contain
  *entropy* from identity secrets; weaker threat models that exclude this are
  better for modelling attacks on the RNG than on memory; see [11SSEK]_.

Corrupt member (of some targets)
  This may be either a genuine but malicious member, or a external attacker
  that has exploited the running system of a genuine member. Unfortunately, it
  is probably impossible to distinguish the two cases.

  We include this for completeness as "the worst case", though it's unclear if
  this is fundamentally different from many repeated applications of "leak
  session secrets"; more research is needed here. One difference could be that
  under the corruption attack, there is no parallel honest instance *and* the
  attacker can observe secret computations that don't touch memory, e.g.
  collecting, using, then immediately discarding entropy.

Now, we enumerate all the concrete things an attacker *might* be able to do, by
applying these attacks to our model of session mechanics. For simplicity, we
only consider individual attacks; a full precise formal treatment will need to
consider multiple attacks across multiple sessions. In the following, the term
"target" refers to the direct target of a secrets-leak or corruption, and the
term "current" refers to when that attack happens.

Older sessions (i.e. already-closed)
  - read session events (i.e. decrypt/verify messages and membership changes);
    [#uenc]_ [#fwds]_
  - (participation is not applicable, since the session is already closed).

Current sessions (i.e. opened, not-yet-closed)
  - read session events; [#fwds]_
  - participate as targets (i.e. auth/encrypt messages and initiate/confirm
    membership changes; this includes making false claims or omissions in the
    *contents* of messages, such as receipt acknowledgements, which may break
    other properties like authenticity of ordering or message membership, or
    invariants on the application layer);
  - participate as non-targets (e.g. against the targets); [#kcis]_
  - (these may apply differently to newer or older parts of the session).

Newer sessions (i.e. not-yet-opened)
  - open/join sessions as targets, read their events and participate in them;
  - open/join sessions (etc.) as non-targets (e.g. invite a target or intercept
    or respond to their invitations), read and participate in them; [#kcis]_
  - read session events, participate as targets, or as non-targets, in sessions
    whose establishment was not compromised as in the previous points. [#rene]_

Next, we'll discuss the unavoidable consequences of attacks using these powers,
and from this try to get an intuitive idea on the best thing we *might* be able
to defend against:

Active attack
  Present cryptographic systems have security theorems that state that, when
  implemented correctly and assuming the hardness of certain mathematical
  problems, an attacker at this level cannot break the confidentiality and
  authenticity of message contents, or the authenticity of membership -- and
  therefore also anything that derives their security from these properties.

  However, side channel attacks against the confidentiality of membership are
  feasible; hiding this sufficiently well (and even defining models for all of
  the side channels) is still a research problem, and it is not known what the
  maximum possible protection is.

Leak identity secrets (and active attack)
  By definition, the attacker may open/join sessions *as targets*, then read
  their events and participate in them. However, we may be able to prevent them
  from doing anything else.

Leak session secrets (and active attack)
  By definition, the attacker may participate *as targets* in current sessions,
  and read its events, for at least several messages into the future -- until
  members mix in new entropy secret to the attacker. If session secrets also
  include identity secrets, the attacker also gains the abilities mentioned in
  the previous point. However, we may be able to prevent them from doing
  anything else.

Corrupt insider (and active attack)
  This is similar to the above, except that there is no chance of recovery by
  members mixing in entropy, until after the corruption is healed.

As just discussed, under certain attacks we can't protect confidentiality, even
for actions of members not directly targeted by that attack. Such is the nature
of our group session model. But we *could* try to protect a related property:

**Confidential authenticity**
  Only the intended consumers should be able to verify the information. (Of
  course, attackers who break confidentiality, may choose to believe the
  information even without being able to verify it.)

For every type of information where we want authenticity *and* confidentiality,
we should also aim for confidential authenticity -- if the attacker should be
unable to read its contents, they should also be unable to verify *anything*
about it. As noted earlier, this is useful not merely for its own sake, but is
essential if we want to protect the confidentiality of membership. Furthermore,
this property can be removed on a higher layer, e.g. an opt-in method to sign
all messages with public signature keys, but once lost it cannot be regained.
So it is safer to default to *having* this property.

Against an active attacker, this means that verification must be executable
only by other members, i.e. depend on session secrets. Against a corrupt
insider, who is already allowed to perform verification, this means we must
choose a deniable or zero-knowledge authentication mechanism, so that they are
at least unable to pass this certainty-in-belief onto third parties.

.. [#uenc] It may be theoretically possible to restrict this only to messages
    authored *by non-targets* (excluding messages that the *target* sends), and
    likewise for membership changes. However, it's unlikely that the complexity
    of any solution is worth the benefit, since (a) for every member, we'd need
    to arrange that they can't derive the ability to decrypt their own messages
    from their session secrets, and (b) even with this protection, the attacker
    can still just compromise a second member to get the missing pieces.

.. [#fwds] The inability for an attacker to decrypt past messages is commonly
    known as *forward secrecy*. Currently are several slightly different formal
    models for this, but the general idea is the same.

.. [#kcis] This attack is commonly known as *key compromise impersonation*.
    As with forward secrecy, there are slightly different models for this.

.. [#rene] For example, the protocol may offer the ability for members to check
    out-of-band, after establishment, that shared session secrets match up as
    expected, and if so then be convinced that the session has full security
    (i.e. session establishment was not tampered with) *even if members know*
    that identity secrets were already compromised before establishment.
