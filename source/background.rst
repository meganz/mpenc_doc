.. include:: <isotech.txt>
.. include:: <mmlalias.txt>

==========
Background
==========

This chapter is not about our progress, but a general discussion of secure
group communications - our model of what it is, and "ideal" properties that
*might* be achieved. Often, lists of security properties may seem arbitrary,
with long names seemingly unrelated to each other. We take a more methodical
approach, and try to classify these properties within a general framework.

Our project goals only focus on a subset of these. Even so, enumerating all the
possibilities that we can think of, is useful for future reference and for
comparison with other projects that focus on a different subset. It gives us
some level of confidence that we haven't missed anything, provides a better
understanding of the relationships between various properties, and suggests a
path towards a more complete and precise framework in the future.

Model and mechanics
===================

To start with, we present a high-level conceptual model of a *private group
session* and introduce some terminology.

Secure communication systems generally proceeds as follows:

0. All members know others' identity (long-term) public keys.
1. Session membership change, inc. establishment and optional teardown.
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
other. Events are only visible to those who are part of the session membership
of the event generator, when they generated it. For example, joining members
cannot read messages that were written before the author saw them join (unless
an old member leaks the contents to them, outside of the protocol).

On the transport level, we assume an efficient packet broadcast operation, that
costs time near-constant in the number of recipients, and bandwidth near-linear
(or less) in the number of recipients or the size of the packet. Note that the
*sender* and *recipients* of a transport packet are concepts distinct from the
author and readers of a logical message; our choice of terminology tries to
make this clear and unambiguous.

For completeness, we note that a real system often unintentionally generates
"side channel" information, beyond the purpose of the model. This includes:

- time, space, energy cost of processing
- size, location of storage
- time, size, route of communications

and probably more that we've missed. This is sometimes unavoidable, but not
always. We'll keep this in mind for later.

We've enumerated the types of information that we have - membership, ordering,
content, and side channel data. Next, we'll discuss how each of the abstract
properties (from the previous section) apply to each of these types.

.. [#keyv] for example, "WARNING: the authenticity and privacy of this session
    is dependant on the unknown validity of the binding $key |LeftRightArrow|
    $user" or perhaps something less technical.

Security properties
===================

What security properties do we want? We consider this in an abstract way, then
methodically apply our conclusions to different specific cases.

In any system, we produce and consume information. This could be explicit (e.g.
contents) or implicit (e.g. metadata). From this very general observation, we
can suggest a few fundamental security properties:

**Efficiency**.
  *Nobody* should be able to cause anyone to spend resources (i.e. time,
  memory, or bandwidth) much greater than what the initiator spent.

**Authenticity**
  Information should be associated with a proof, either explicit or implicit,
  that consumers may use to verify the truth of it.

**Confidentiality**
  Only the intended consumers should be able to access and interpret the
  information.

We'll deal with efficiency in more specific terms later. For now, we discuss
how authenticity and confidentiality apply to the information types from above.
Even though we don't try to achieve all of these, being aware of them enables
us to avoid decisions that destroy the possibility to achieve them in future.

+-------------------+-------------------+-------------------+
|                   | Authenticity      | Confidentiality   |
+===================+===================+===================+
| Existence         | auto              | research          |
+-------------------+-------------------+-------------------+
| Side channels     | unneeded          | research          |
+-------------------+-------------------+-------------------+
| Membership        | crypto            | research          |
+-------------------+-------------------+-------------------+
| Ordering          | crypto            | unknown           |
+-------------------+-------------------+-------------------+
| Contents          | crypto            | crypto            |
+-------------------+-------------------+-------------------+

Auth. of session existence
  This is achieved automatically by authenticity of any of the other types; we
  don't need to worry about it on its own.

Conf. of session existence
  This is the hardest to achieve, and is on ongoing research topic. Not only
  does it require confidentiality of all the other types of information, but
  also trusted transport obfuscation and/or steganography. (because?)

Auth. of side channels
  We don't care about the authenticity of something we didn't intend to
  communicate in the first place.

Conf. of side channels
  Still a research topic, this is a concern because it may be used to break
  the confidentiality of other types of information.

Auth. of membership
  There is some depth to this. The first choice is whether a distinct "change
  session membership" operation should exist outside of sending messages. "Yes"
  means that (e.g.) you can add someone to the session, and they will know this
  (maybe a window will pop up on their side) even if you don't send them any
  messages. "No" means that membership changes must always be associated with
  an actual message that effects this change. This is up to the application;
  though "no" is generally more suited for asynchronous scenarios.

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

Threat models
=============

Attackers with different powers may try to break any of the above properties.
Firstly, let's define and describe these powers:

Active communications attack (on the entire transport)
  This is the standard attack that all modern communications systems should
  protect against - i.e. a transport-level "man in the middle" that can inject,
  drop, replay and reorder packets. Generally (and in practise mostly) channels
  are bi-directional - so we must assume that the attacker, if they are able to
  target one member, then they are able to target all members, by attacking the
  channel in the opposite direction.

Leak identity secrets (of some targets)
  This refers to all of the secret material needed for a subject to establish a
  new session, e.g. with someone that they've never communicated with before.
  By definition, this is secret only to the subject.

Leak session secrets (of some targets)
  This refers to all of the secret material needed for a subject to continue
  participating in a session they're already part of. This may be secret to
  only the subject, or shared across all members, or a combination of both.

  Depending on the protocol, this may *include* identity secrets if they must
  be used to generate outgoing or process incoming messages. Better protocols
  would *not* need this, so that these may be wiped from memory during the
  session, reducing the attack surface.

Corrupt member (of some targets)
  This may be either a genuine but malicious member, or a external attacker
  that has exploited the running system of a genuine member. Unfortunately, it
  is probably impossible to distinguish the two cases.

  We include this because it's commonly mentioned, but it's unclear whether
  this is fundamentally different from many repeated applications of "leak
  session secrets"; more research is needed here. One difference could be that
  under the corruption attack, there is no parallel honest instance *and* the
  attacker can observe secret computations that don't touch memory, e.g.
  collecting, using, then immediately discarding entropy.

Next, we'll discuss the unavoidable consequences of attacks using these powers,
and from this try to get an intuitive idea on the best thing we *might* be able
to defend against:

Active attack
  Present cryptographic systems have security theorems that state that, when
  implemented correctly and assuming the hardness of certain mathematical
  problems, an attacker at this level is unable to break the conf. and auth. of
  message contents, or the auth. of membership - and therefore also anything
  that derives their security from these mechanisms.

  However, side channel attacks against the conf. of membership is still
  feasible; hiding this sufficiently well (and even defining models for all of
  the side channels) is still a research problem, and it is not known what the
  maximum possible protection is.

Leak identity secrets (+ active):
  By definition, the attacker may initiate sessions or confirm invitation *as
  the target*. However, perhaps we might be able to defend against the attacker
  trying to:

  - start new sessions or confirm invitations *as others* (e.g. inviting the
    target or intercepting the target's invitations);
  - decrypt/verify messages from older sessions;
  - participate in current sessions, or newer sessions that are established
    without the attacker's interference.

Leak session secrets (+ active):
  By definition, the attacker may participate *as the target* in current
  sessions - for at least several messages, until the members mix in entropy
  secret to the attacker. This means that they may:

  - authenticate/encrypt messages or write membership changes, as the target;
  - decrypt/verify messages or read membership changes, sent to the session;
    [#uenc]_
  - make false claims or omissions in the *contents* of messages, which may
    break other concerns like auth. of ordering, or application invariants.

  However, perhaps we might be able to defend against the attacker trying to:

  - exercise those capabilities in the current session *as others*;
  - exercise those capabilities in older sessions;
  - (if session secrets don't include identity secrets) initiate or confirm
    invitation to, or exercise those capabilities in, newer sessions.

Corrupt insider (+active):
  This is similar to the above, except that there is no chance of recovery by
  members mixing in entropy, until after the corruption is healed.

So under certain attacks we can't protect confidentiality, even for actions of
members that aren't compromised. Such is the nature of our group session model.
But we *could* try to protect a related property:

**Confidential authenticity**
  Only the intended consumers should be able to verify the information. (Of
  course, attackers who break confidentiality, may choose to believe the
  information even without being able to verify it.)

As noted earlier, this is useful not merely for its own sake, but is essential
if we want to protect the confidentiality of membership. Against an active
attacker, this means that verification must be executable only by other members
of the session, i.e. depend on session secrets. Against a corrupt insider, who
is already allowed to verify content authorship, this means we must choose a
deniable or zero-knowledge authentication mechanism, so that they are at least
unable to pass this certainty-in-belief onto third parties.

.. [#uenc] It may be theoretically possible to restrict this only to messages
    sent *by others* (i.e. excluding messages that the *target* sends), and
    likewise for membership changes. However, it's unlikely that the complexity
    of any solution is worth the benefit, since (a) for every member, we'd need
    to arrange that they can't derive the ability to decrypt their own messages
    from their session secrets, and (b) even with this protection, the attacker
    can still just compromise a second member to get the missing pieces.
