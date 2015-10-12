TODO: review this and add references

*******************************************
Authenticated Signature Key Exchange (ASKE)
*******************************************

As previously noted, the user as well as the user's messages have to
be authenticated.  This could easily be accomplished by using the
static private keys, however this would not be beneficial for three
reasons: The high computational overhead on the client of discrete
logarithm public-key cryptography, the significantly increased message
sizes, and lastly a plausible deniability aspect.

Particularly the last reason deserves some further explanation.  Even
though we are (at this time) not worried about ensuring deniability
for the encryption protocol, it is easy to achieve a certain degree of
deniability of the messages sent by generating an ephemeral signing
key pair at the beginning of the chat session.  At the end of the chat
session the ephemeral private signing key is published publicly to the
group, allowing for retrospective chat transcript alteration, and
therefore creating a state of plausible deniability for the messages
sent and/or received.  However, the mechanism outlined *does not*
provide deniability on the participation in the authenticated
signature key exchange (ASKE) protocol.

The approach to derive a contributary session ID from participants'
nonces to ensure freshness, as well as retrospective authentication of
the session at the end of the agreement has been inspired by [06DGKA]_.
Van Gundy has also based his "Improved Deniable Signature Key Exchange
for mpOTR" [13DSKE]_ on an extension of [06DGKA]_.  As the CLIQUES
protocol requires an upflow (sequential collect) and downflow
(broadcast) structure (see :doc:`crypto_gka`), whereas
[06DGKA]_ and [13DSKE]_ are based on a constant-round broadcast
protocol.  To make the overall combined protocol flow more efficient,
(combined key agreement messages, see :doc:`crypto`), the
commitment phase (see below) messages have been sequentialised into a
collective upflow.  The first broadcast of the last member in the
chain is the start of the downflow, which (unlike the group key
agreement) is followed by an acknowledgement broadcast by every other
participant.

ASKE consists of three phases.  First each participant generates a
nonce and an ephemeral signature key pair, and forwards the nonce and
public key (upflow).  In the second phase -- when in possession of all
nonces -- each member independently computes the shared session ID,
and authenticates their ephemeral signing key and session ID using
their static private key (downflow).  Lastly, each participant
verifies all received acknowledgement messages.

The protocol is designed to authenticate a user's ephemeral signing
key indirectly against their long term/static key while preventing
replay attacks (ensures freshness) through the way the session ID is
computed.  In case of using DSA or RSA for the long term secret, this
approach keeps message sizes and computational overhead low
(particularly if implemented on slower devices or in JavaScript).


Phase 1 -- Commitment
=====================

In the first phase a participant :math:`i` with the participant ID
:math:`pid_i` commits to participate in the group chat.  For this the
session initiator compiles an ordered list of all group members (their
participant IDs).  Additionally an empty list for the all
participants' nonces and ephemeral public keys is initialised.

The first participant then generates randomly a nonce :math:`k` and an
ephemeral private/public key pair :math:`(e, E)`. Both of these are
added to the nonces and public keys lists. The participants' list
(:math:`pid_i`), nonces list (:math:`k_i`) and public keys list
(:math:`E_i`) are then sent on to the next member in the participants'
list, who again generates a nonce and ephemeral signature key pair to
send on.

This initial commit phase (upflow) terminates with the last member in
the list to add their contributions.  This last member is the
initiator of the second phase.

**Example:**

The following figure shows the sequence of upflow messages
(:math:`uX`) sent among four participants (:math:`pX`).

.. digraph:: aske_upflow
   :caption: Sequential, directed messages sent during the commitment
             (upflow) phase.

   layout=circo;
   size=2;
   node [style=filled, shape=circle];
   p1 -> p2 [arrowhead=empty, color=blue, label="u1"];
   p2 -> p3 [arrowhead=empty, color=blue, label="u2"];
   p3 -> p4 [arrowhead=empty, color=blue, label="u3"];
   p4 -> p1 [style=invis];

Message contents:

* :math:`u1`:

   * Participants: :math:`(p1,\; p2,\; p3,\; p4)`
   * Nonces: :math:`(k_1)`
   * Ephemeral public signing keys: :math:`(E_1)`

* :math:`u2`:

   * Participants: :math:`(p1,\; p2,\; p3,\; p4)`
   * Nonces: :math:`(k_1,\; k_2)`
   * Ephemeral public signing keys: :math:`(E_1,\; E_2)`

* :math:`u3`:

   * Participants: :math:`(p1,\; p2,\; p3,\; p4)`
   * Nonces: :math:`(k_1,\; k_2,\; k_3)`
   * Ephemeral public signing keys: :math:`(E_1,\; E_2,\; E_3)`

.. _aske-session-sig:

Phase 2 -- Acknowledgement
==========================

TODO(xl): we probably also want to hash group_key in here, to detect members
tampering with the group key e.g. giving different results to different members.

In the acknowledgement phase every member who is in possession of
complete lists of participants' contributions confirms their ephemeral
key and the session.  For this, a session ID :math:`sid` is generated
from all participant IDs and nonces using a hash function :math:`H`:

.. math::
   sid = H(pid_1||pid_2||\ldots||pid_n||k_1||k_2||\ldots||k_n)

A strict ordering criterion has to be used for ordering of the IDs and
nonces (order in list).  For mpENC for the Mega platform the
participant IDs are the full XMPP JIDs, and sorting is performed in
lexical order.  The nonces are ordered to align in their order with
their corresponding participant IDs.

The initiator of the downflow in the acknowledgement phase generates
an authenticator message using their own contributions:

.. math::
   m_i = (magic\:number||pid_i||E_i||k_i||sid)

Here, :math:`magic\:number` is a fixed string to distinguish the
authenticator from any other content (for now it is the byte sequence
"``acksig``"). A broadcast message contains the now completed lists of
participants (:math:`pid_i`, for all :math:`i`), nonces (:math:`k_i`,
for all :math:`i`) and public keys (:math:`E_i`, for all :math:`i`)
along with a signature of their own authenticator message
:math:`\sigma_{s_i}(m_i)` (computed with the static signature key pair
:math:`(s, S)`).  This broadcast message is sent to all participants.

Every participant is in possession of the information required to
produce each participant's :math:`m_i` and verify its signature
:math:`\sigma_{s_i}(m_i)`.  The purpose is to authenticate a
particular ephemeral signing key for one participant in a specific
session.  Adding the nonce :math:`k_i` to :math:`m_i` may strictly not
be necessary (as it is an element already used for computing
:math:`sid`), but it should not matter much to include it as well.

Upon receipt of such a acknowledgement message each participant who
has not acknowledged yet, will likewise send such a broadcast message,
now that all required information is available.

**Example:**

The following figure shows the corresponding downflow message
(:math:`d4`) broadcast to all participants by :math:`p4`.

.. digraph:: aske_downflow
   :caption: First broadcast message sent during the acknowledgement
             (downflow) phase.

   layout=circo;
   size=2;
   node [style=filled, shape=circle];
   p1 -> p2 [style=invis];
   p2 -> p3 [style=invis];
   p4 -> {p1 p2 p3} [label="d4"];

Message content of :math:`d4` from participant 4:

   * Participants: :math:`(p1,\; p2,\; p3,\; p4)`
   * Nonces: :math:`(k_1,\; k_2,\; k_3,\; k_4)`
   * Ephemeral public signing keys: :math:`(E_1,\; E_2,\; E_3,\; E_4)`
   * Session signature: :math:`\sigma_{s_4}(m_4)`

Upon receipt of :math:`d4` every other participant sends out an
analogous :math:`dX` message including their *own* session signature.


.. _ASKE_verification:

Phase 3 -- Verification
=======================

This last phase does not require further messages to be sent.  Each
participant verifies the content of each received acknowledgement
broadcast message against their own available information.  The
purpose behind this is to have the assurance that all participants are
actively participating (avoids replays) with a fresh session, and to
have the assurance that the session's ephemeral signing keys are
genuinely from the users one is communicating with.  In the following
*only* the ephemeral keys are needed for message authentication,
whereas signing with the static keys would effectively inhibit any
plausible deniability.

The verification process:

* Compute the session ID (:math:`sid`) from the content of the
  received broadcast message.

* If a previously self-computed session ID (:math:`sid`) is available
  already, compare it to the one computed from the message content.

* Compute locally the broadcast message sender's :math:`m_i`, and
  verify it against its signature :math:`\sigma_{s_i}(m_i)` received
  using the sender's long term static key :math:`S_i`.

In case any verification above fails, an mpENC error message with
``TERMINAL`` severity must be broadcast to inform all participants of
the failure.


Auxiliary Protocol Runs
=======================

Upon change in the participant composition of the chat (joins or
exclusions of members) some session information changes: The list of
participants, nonces and ephemeral signing keys.  Therefore, also the
session ID :math:`sid` changes.


Member Addition (join)
----------------------

On joining participants an initiator is extending the list of
participants by the new participant(s).  A new commitment (upflow)
message is sent to the (first) new participant, including the *new*
list of participants :math:`p_i` and *already existing* nonces
:math:`k_i` and ephemeral signing keys :math:`E_i`.  The commitment
upflow percolates through all new participants, and the last one will
initiate a new acknowledgement downflow phase followed by a
verification phase identically to the initial protocol flow as
outlined above.

**Example:**

The following figure shows addition of a participant (:math:`p5`) --
initiated by :math:`p1` -- to the existing group of four participants.

.. digraph:: aske_join
   :caption: Messages involved in an auxiliary ASKE protocol flow for
             the addition of a single participant.

   layout=circo;
   size=2;
   ordering=out;
   node [style=filled, shape=circle];
   p5 [style=dashed];
   p1 -> p2 -> p3 -> p4 [style=invis];
   p1 -> p5 [arrowhead=empty, color=blue, label="u1'"];
   p5 -> {p1 p2 p3 p4} [label="d5'"];

Message contents:

* :math:`u1'`:

   * Participants: :math:`(p1,\; p2,\; p3,\; p4,\; p5)`
   * Nonces: :math:`(k_1,\; k_2,\; k_3,\; k_4)`
   * Ephemeral public signing keys: :math:`(E_1,\; E_2,\; E_3,\; E_4)`

* :math:`d5'`:

   * Participants: :math:`(p1,\; p2,\; p3,\; p4,\; p5)`
   * Nonces: :math:`(k_1,\; k_2,\; k_3,\; k_4,\; k_5)`
   * Ephemeral public signing keys: :math:`(E_1,\; E_2,\; E_3,\; E_4,\; E_5)`
   * Session signature: :math:`\sigma_{s_5}(m_5)`

After receiving this message, :math:`p1` through :math:`p4` will
likewise broadcast their acknowledgement messages to all participants
as well as verify all received session signatures
:math:`\sigma_{s_i}(m_i)`.


Member Exclusion
----------------

On member exclusion, the process is simpler as it does not require a
commitment (upflow) phase, as all remaining participants have
committed already.  The initiator of the exclusion removes the
excluded participant(s) from the list of participants, and their
respective nonces and ephemeral signing keys are as well removed.
Additionally the initiator will update their own nonce to prevent
collisions in the session ID :math:`sid` with a previous session ID
consisting of the same set of participants.  From these updated values
a new session ID :math:`sid` and session signature
:math:`\sigma_{s_i}(m_i)` is computed.  This updated information is
then used to directly broadcast the acknowledgement downflow message
to all remaining participants.  Each of the remaining participants
again validates all received signatures :math:`\sigma_{s_i}(m_i)` and
broadcasts their own acknowledgement (if still outstanding).


Member Departure
----------------

Member departure is the "voluntary" parting of a participant rather
than an exclusion through another participant.  In effect it is the
same, with the only difference that the departing member indicates the
desire to leave, and a member exclusion auxiliary protocol run will be
initiated upon that by another participant (the owner).  To support
plausible deniabilty, a departing member should include their
ephemeral signing key in the farewell message.


Key Refresh
-----------

The concept of a key refresh for ASKE is currently not considered.


..
    Local Variables:
    mode: rst
    ispell-local-dictionary: "en_GB-ise"
    mode: flyspell
    End:
