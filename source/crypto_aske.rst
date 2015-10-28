===========================================
Authenticated Signature Key Exchange (ASKE)
===========================================

As previously noted, we must authenticate the user's membership in the session,
as well as authenticate the content of their messages.  This could easily be
accomplished by using static private keys to make signatures.  However, this
destroys any chance that we might have in the future at retaining deniability
(also known as repudiability) of ciphertext.  Once this property is lost, we
can never regain it in a higher layer, and it is critical for *confidentiality
of metadata*, so it is better to retain it here.

Instead, we generate ed25519 ephemeral signing keys for each participant, to be
used to authenticate messages for the current session only.  We also derive a
session ID from participants' nonces to ensure freshness.  At the end of the
session the ephemeral private signing key may be published to the transport,
allowing for retrospective chat transcript alteration, and therefore allowing
repudiation of the contents of transcripts presented post-session.  However,
our mechanism *does not* provide deniability of the *participation* in the
authenticated signature key exchange (ASKE) protocol.

Our approach takes some loose inspiration from [06DGKA]_ and [13DSKE]_, and may
also be compared with [03SPAG]_ and [12SDGK]_.  Both are constructions to turn
an unauthenticated GKA into an authenticated one.  The former construction is
not deniable, whereas the latter is fully deniable (including participation in
the session) but requires more advanced cryptography (ring signatures) that
have not yet seen widespread usage.

Our :doc:`GKA <crypto_gka>` protocol consists of an upflow (sequential collect)
and downflow (broadcast) phase, whereas our ASKE protocol is a constant-round
broadcast protocol, as are the other authenticated agreements referenced above.
To make our :doc:`combined protocol <crypto>` easier to understand, the first
broadcast round has been serialised into a "collection phase" upflow.  The
first broadcast of the last member in the chain is the start of the downflow,
which is followed by an acknowledgement broadcast by every other participant.
Note that these latter broadcasts are not present in our group key agreement.

ASKE consists of three phases.  First each participant generates a nonce and an
ephemeral signature key pair, and forwards the nonce and public key (upflow).
In the second phase -- when in possession of all nonces -- each member
independently computes the shared session ID, and authenticates their ephemeral
signing key and session ID using their static private key (downflow).  Lastly,
each participant verifies all received acknowledgement messages.


Initial Protocol Run
====================

Phase 1 -- Collection
---------------------

In the first phase the session initiator :math:`i` with the participant ID
:math:`\mathsf{pid}_i` compiles an ordered list of all group members (their
participant IDs).  Additionally an empty list for the all participants' nonces
and ephemeral public keys is initialised.

The initiator then generates a nonce :math:`k_i` and an ephemeral signature key
pair :math:`(e_i, E_i)`.  They add these to the nonces and public keys lists.
They then send the participants' list (:math:`\mathsf{pid}_i`), nonces list
(:math:`k_i`) and public keys list (:math:`E_i`) on to the next member in the
list, who again generates a nonce and ephemeral signature key pair to send on.

This phase ends with the last member in the list to add their contributions.
This last member is the initiator of the second phase.

**Example:**

The following figure shows the sequence of upflow messages
(:math:`u_i`) sent among four participants (:math:`p_i`).

.. digraph:: aske_upflow

   layout=circo;
   size=2;
   node [style=filled, shape=circle];
   p1 -> p2 [arrowhead=empty, color=blue, label="u1"];
   p2 -> p3 [arrowhead=empty, color=blue, label="u2"];
   p3 -> p4 [arrowhead=empty, color=blue, label="u3"];
   p4 -> p1 [style=invis];

:math:`u1` contains:
   * Participants: :math:`(p1,\; p2,\; p3,\; p4)`
   * Nonces: :math:`(k_1)`
   * Ephemeral public signing keys: :math:`(E_1)`

:math:`u2` contains:
   * Participants: :math:`(p1,\; p2,\; p3,\; p4)`
   * Nonces: :math:`(k_1,\; k_2)`
   * Ephemeral public signing keys: :math:`(E_1,\; E_2)`

:math:`u3` contains:
   * Participants: :math:`(p1,\; p2,\; p3,\; p4)`
   * Nonces: :math:`(k_1,\; k_2,\; k_3)`
   * Ephemeral public signing keys: :math:`(E_1,\; E_2,\; E_3)`

.. _aske-session-sig:

Phase 2 -- Acknowledgement
--------------------------

The initiator of the downflow in the acknowledgement phase first constructs an
authenticator message from their own contributions:

.. math::
   m_i = (\mathsf{CTX}||\mathsf{pid}_i||E_i||k_i||\mathsf{sid})

Here, :math:`\mathsf{CTX}` is a fixed byte sequence to prevent the signature
being used in another application; for this protocol version we use ``acksig``.
:math:`\mathsf{sid}` is the *session ID*, calculated from all participant IDs
and nonces using a hash function :math:`H`:

.. math::
   \mathsf{sid} = H(\mathsf{pid}_1||\mathsf{pid}_2||\ldots||\mathsf{pid}_n||k_1||k_2||\ldots||k_n)

The IDs and nonces must be strictly ordered.  For mpENC on the Mega platform
the participant IDs are the full XMPP JIDs, and sorting is performed in lexical
order.  The nonces are ordered so as to correspond to their participant IDs.

Then, the initiator broadcasts the first message in the downflow, containing
the now-completed lists of participants (:math:`\mathsf{pid}_i`), nonces
(:math:`k_i`) and public keys (:math:`E_i`), for all :math:`i`, along with a
signature of their own authenticator message :math:`\sigma_{s_i}(m_i)` computed
with the static identity signature key :math:`(s_i, S_i)`.  The purpose of this
is to authenticate all information contributed by the signing participant, as
well as what they believe the contributions of all session members to be.
[#maut]_  Note that the authenticator message itself needs not be, and is not,
broadcast.

After receiving this, every participant is in possession of the information
required to calculate the supposed :math:`\mathsf{sid}` for themselves, produce
what each :math:`m_i` *should be* and verify the :math:`\sigma_{s_i}(m_i)` that
it *should have* based on this information.

Now, each participant computes the session ID (:math:`\mathsf{sid}`) from the
content of this initial broadcast message, checking that the values supposedly
contributed by them actually match what they output during the upflow phase.
Then, they generate their own authenticator message, corresponding signature,
and broadcast this signature to others.  The lists of intermediate values are
not necessary in these further broadcasts.

**Example:**

The following figure shows the corresponding downflow message
(:math:`d4`) broadcast to all participants by :math:`p4`.

.. digraph:: aske_downflow

   layout=circo;
   size=2;
   node [style=filled, shape=circle];
   p1 -> p2 [style=invis];
   p2 -> p3 [style=invis];
   p4 -> {p1 p2 p3} [label="d4"];

:math:`d4` contains:
   * Participants: :math:`(p1,\; p2,\; p3,\; p4)`
   * Nonces: :math:`(k_1,\; k_2,\; k_3,\; k_4)`
   * Ephemeral public signing keys: :math:`(E_1,\; E_2,\; E_3,\; E_4)`
   * Session signature: :math:`\sigma_{s_4}(m_4)`

Upon receipt of :math:`d4` every other participant sends out an
analogous :math:`dX` message including their *own* session signature.

.. [#maut] Although :math:`k_i` is already "contained in" the session ID, we
    explicitly add it to :math:`m_i`, to avoid depending on the security of its
    calculation. This is hoped to simplify any future formal analysis.


.. _ASKE_verification:

Phase 3 -- Verification
-----------------------

This last phase does not require further messages to be sent.  Each participant
verifies the content of each received acknowledgement broadcast message against
their own available information.  The purpose is to have the assurance that all
participants are actively participating (avoids replays) with a fresh session,
and to have the assurance that the session's ephemeral signing keys are really
from the users that one is communicating with.

More specifically, as each participant receives each subsequent downflow
broadcast from :math:`\mathsf{pid}_i`, they compute :math:`m_i` from the same
information used to compute their local value for :math:`\mathsf{sid}`, and
verify the signature contained in the received message (which is supposed to be
:math:`\sigma_{s_i}(m_i)`) using the sender's long term static key :math:`S_i`.

The protocol completes successfully when all session signatures from all other
participants have been successfully verified against the local session ID and
each participant's static identity signature key.

Following successful completion, *only* the ephemeral keys are needed for
message authentication -- signing with the static keys would effectively
inhibit any plausible deniability.  However the static keys are needed for
further changes to the session membership.


Auxiliary Protocol Runs
=======================

Upon changing the participant composition of the chat (inclusions or exclusions
of members) some session information changes: The list of participants, nonces
and ephemeral signing keys.  Therefore, the session ID also changes.


Member Inclusion
----------------

To include participants, the initiator extends the list of participants by the
new participant(s).  A new collection (upflow) message is sent to the (first)
new participant, including the *new* list of participants :math:`p_i` and
*already existing* nonces :math:`k_i` and ephemeral signing keys :math:`E_i`.
The collection upflow percolates through all new participants, and the last one
will initiate a new acknowledgement downflow phase followed by a verification
phase identically to the initial protocol flow as outlined above.

**Example:**

The following figure shows addition of a participant (:math:`p5`) -- initiated
by :math:`p1` -- to the existing group of four participants.

.. digraph:: aske_include

   layout=circo;
   size=2;
   ordering=out;
   node [style=filled, shape=circle];
   p5 [style=dashed];
   p1 -> p2 -> p3 -> p4 [style=invis];
   p1 -> p5 [arrowhead=empty, color=blue, label="u1'"];
   p5 -> {p1 p2 p3 p4} [label="d5'"];

:math:`u1'` contains:
   * Participants: :math:`(p1,\; p2,\; p3,\; p4,\; p5)`
   * Nonces: :math:`(k_1,\; k_2,\; k_3,\; k_4)`
   * Ephemeral public signing keys: :math:`(E_1,\; E_2,\; E_3,\; E_4)`

:math:`d5'` contains:
   * Participants: :math:`(p1,\; p2,\; p3,\; p4,\; p5)`
   * Nonces: :math:`(k_1,\; k_2,\; k_3,\; k_4,\; k_5)`
   * Ephemeral public signing keys: :math:`(E_1,\; E_2,\; E_3,\; E_4,\; E_5)`
   * Session signature: :math:`\sigma_{s_5}(m_5)`

After receiving this message, :math:`p1` through :math:`p4` will likewise
broadcast their acknowledgement messages to all participants as well as verify
all received session signatures :math:`\sigma_{s_i}(m_i)`.


Member Exclusion
----------------

On member exclusion, the process is simpler as it does not require a collection
(upflow) phase, as all remaining participants have announced already.  The
initiator of the exclusion removes the excluded participant(s) from the list of
participants, and their respective nonces and ephemeral signing keys are as
well removed.

Importantly, the initiator *must* update their own nonce to prevent collisions
in the session ID :math:`\mathsf{sid}` with a previous session ID consisting of
the same set of participants.  They then compute a new session ID and session
signature :math:`\sigma_{s_i}(m_i)` from these updated values, and used them to
directly broadcast the initial downflow message to all remaining participants.
Each of them again verifies all session signatures and broadcasts their own
acknowledgement (if still outstanding).


Key Refresh
-----------

The concept of a key refresh for ASKE is currently not considered.


Member Departure
================

As with our GKA, our ASKE does not include a *member departure* operation, this
instead being handled in a different part of the wider protocol.

In the future, our departure mechanism will include publishing the ephemeral
signature key, to support a limited form of ciphertext deniability.  This is
described more in :ref:`Publish signature keys <publish-sess-sig-keys>`.


Confirmation of the Shared Group Key
====================================

The ASKE mentioned above only protects the GKA against external attackers.  A
malicious insider can cause different participants to generate different shared
group keys.  We *could* protect against this by adding the shared group secret
:math:`x_1 \dots x_nG` (or better, something derived from it) to the definition
of :math:`m_i`. However, we currently omit this, for the following reasons:

- This would cause a key refresh to require everyone's acknowledgement, adding
  an extra round to it.

- This attack does no more damage beyond "drop all messages", which is already
  available to active transport attackers:

  - For authenticity of session membership, i.e. *entity authentication* or
    *freshness*, all participants are indeed part of the session, having
    participated in the ASKE and verified each others' session signatures. They
    just may have generated different encryption keys.

  - For authenticity of message content: each ephemeral signature includes a
    hash of something derived from the group key, so one will never correctly
    verify-decrypt a message that was encrypted using a different group key
    from yours.

  - For authenticity of message membership: our other checks (for reliability
    and consistency) will cause each side of a "split" to timeout and emit
    security warnings, because the other side was unable to acknowledge them.

However, it is not too much complexity to add such a feature, if the attack
turns out to be a real problem.  Another benefit of adding the protection would
be to make the above argument unnecessary, which reduces the complexity cost of
analysing our protocol.


..
    Local Variables:
    mode: rst
    ispell-local-dictionary: "en_GB-ise"
    mode: flyspell
    End:
