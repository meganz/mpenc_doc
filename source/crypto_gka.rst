TODO: review this and add references

*************************
Group Key Agreement (GKA)
*************************

The group key agreement is conducted according to CLIQUES [00CLIQ]_.  The
protocol has only been modified towards using ECDH based on
x25519 instead of classical DH exponentiation.

The protocol allows for a shared group key negotiation for :math:`n`
members in a total of :math:`O(n)` messages sent, and re-negotiations
for (single) joins, excludes or key refreshes of :math:`O(1)` messages
(if broadcasts are available).

Messages for the CLIQUES upflow are co-located with with the ASKE
phase 1 messages (see :doc:`crypto_aske`), and the CLIQUES downflow with ASKE
phase 2 messages.

.. note::
   CLIQUES by itself does not protect against active attackers
   (man-in-the-middle attacks).  However, all messages are co-located
   with the ASKE messages, and *each* message is signed.
   Therefore an active attack is effectively countered by the final
   ASKE verification step (see :ref:`ASKE_verification`).

The CLIQUES key agreement algorithm is based on the
Group-Diffie-Hellman (GDH) concept, in which each participant
generates a private contributory key portion for the session.  This
will be computed into the (public) intermediate keys passed between
the members.  At the end of the process, a set of :math:`n`
intermediate keys will be available to all, with each of them lacking
the private contribution of exactly *one* participant.  Each
participant uses that one respective intermediate key missing their
own contribution to compute the *same* shared group key.

Based on this GDH concept two distinctly different, but related,
protocols are constructed: The initial key agreement (IKA) and the
auxiliary key agreement (AKA).  IKA is used for an initial key
agreement with no initial knowledge of intermediate keys of the other
parties.  AKA is a supplementary and simplified protocol to agree on
an new, changed group key once the IKA protocol has successfully
terminated.  It is executed on joining or excluding participants as
well as refreshing the shared group key.

Besides their slightly different actions within the group key
agreement (e. g. initiator of upflow or of downflow, see below), no
participant of the IKA or AKA bear a special role.  All participants
of the CLIQUES key agreement can be considered equal in terms of the
protocol functionality.  However, it makes sense to consider the role
of the "chat owner", who has the sole privilege to initiate the group
chat's key agreement, as well as joining new participants or excluding
others.  The only AKA action *any* participant may perform is that of
a key refresh.


Initial Key Agreement (IKA)
===========================

The initial key agreement follows the concept as outlined by the
[00CLIQ]_ IKA.1 protocol.  The protocol distinguishes between two
phases: The "upflow" phase and the "downflow" phase.  In the upflow,
each participant generates their private key contribution, and
computes it into the elements of the (growing) chain of intermediate
keys.  Once everyone has participated in the upflow, the last
participant will initiate the downflow by broadcasting the final chain
of intermediate keys to all, enabling them to individually compute the
group key.


Upflow
------

The first participant will assemble a linear list (an ordered chain)
of all participants included in the IKA, being the first member in
that chain.  Then, it generates their private key contribution
(:math:`x_1`), and uses the Diffie-Hellman (DH) concept to compute the
public portion of the key (:math:`g^{x_1}`).  In this case of mpENC,
this will be done using ECDH scalar multiplication with the base point
(:math:`G`): :math:`x_1G`.  The message forwarded to the successor in
the chain contains a list of intermediate keys.  The first one being
"their" intermediate key lacking their own contribution (therefore an
"empty" contribution), followed by the "cardinal key".  The cardinal
key is the most complete key, containing *all* previous participants
contributions (for the first participant in the chain only their own:
:math:`x_1G`).

Successive participants receiving the upflow messages similarly
generate their own private contributions.  This contribution is used
for computing new intermediate keys using DH (ECDH scalar
multiplication with the intermediate keys).  The last element becoming
the new cardinal key (previous cardinal key multiplied with their own
contribution).  "Their" intermediate key is the previously received
cardinal key, which is inserted just before the cardinal key.

**Example:**

The following figure shows the sequence of upflow messages
(:math:`uX`) sent among four participants (:math:`pX`).

.. digraph:: gka_upflow
   :caption: Sequential, directed messages sent during the upflow
             phase.

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
   * Intermediate keys: :math:`(\mathrm{\text{<empty>}},\; x_1G)`

* :math:`u2`:

   * Participants: :math:`(p1,\; p2,\; p3,\; p4)`
   * Intermediate keys: :math:`(x_2G,\; x_1G,\; x_2x_1G)`

* :math:`u3`:

   * Participants: :math:`(p1,\; p2,\; p3,\; p4)`
   * Intermediate keys: :math:`(x_3x_2G,\; x_3x_1G,\; x_2x_1G,\; x_3x_2x_1G)`


Downflow
--------

The last participant in the chain performs the same operations by
adding their own contributions to the intermediate keys.  However, the
cardinal key can already be used to derive the group key.  The group
key is comprised of the first 16 bytes of the hash (SHA-256) of the
last cardinal key.  Therefore, the last participant will retain the
new cardinal key to derive the group key and broadcast the list of
resulting intermediate keys to *all* members for the downflow.  After
this broadcast, this participant has finished their participation in
the IKA protocol and is considered to be initialised and in possession
of the shared, secret group key.

Each recipient of the downflow message will now be able to take
"their" intermediate key out of the list (the one missing their own
contribution).  For the :math:`i`-th member in the chain, this will be
the :math:`i`-th intermediate key.  Through scalar multiplication of
their own private key contribution with "their" intermediate key they
will all derive the same shared, secret group key.  This is the point
this participant has also completed its part in the IKA and has
transitioned into the ready state.

**Example:**

The following figure shows the corresponding downflow message
(:math:`d`) broadcast to all four participants (:math:`pX`).

.. digraph:: gka_downflow
   :caption: Broadcast message sent during the downflow phase.

   layout=circo;
   size=2;
   node [style=filled, shape=circle];
   p1 -> p2 [style=invis];
   p2 -> p3 [style=invis];
   p4 -> {p1 p2 p3} [label="d"];

Message content of :math:`d`:

   * Participants: :math:`(p1,\; p2,\; p3,\; p4)`
   * Intermediate keys: :math:`(x_4x_3x_2G,\; x_4x_3x_1G,\; x_4x_2x_1G,\; x_3x_2x_1G)`

After receiving these intermediate keys, every participant can compute
the same shared group key by multiplying "their" intermediate key with
their own private contribution:

.. math::

   x_1x_4x_3x_2G = x_2x_4x_3x_1G = x_3x_4x_2x_1G = x_4x_3x_2x_1G


Auxiliary Key Agreement (AKA)
=============================

Once an initialised chat encryption is available for an established
group of participants, an auxiliary key agreement (AKA) can be
invoked.  These runs are necessary for changes in group participants
(joining new members or excluding existing ones) to change the group
key.  Therefore allowing the previous participant set only to read
messages before the AKA, and the new participant set to read/write
messages after the AKA.  Furthermore the AKA can be used for
refreshing the group key by updating a participant's private key
contribution.


Member Addition (join)
----------------------

Member addition is performed very similarly to the IKA protocol.  An
existing participant (should be the "chat owner") would initiate an
upflow for this.  To do that, first the new participant(s) are
appended to the list of existing participants.  To avoid the new
participants to gain knowledge of the previous group key, the
initiator of the join is required to update their private key
contribution in the following fashion:

* Perform DH computation (ECDH scalar multiplication) with own private
  contribution on "own" intermediate key (that is the first
  intermediate key in the list in case of the chat owner who has
  started the IKA originally).
* Generate a new private key contribution (see
  :ref:`note_key_contributions`).
* Perform DH computation on all intermediate keys (besides "own") with
  the new private key contribution.

The upflow is now initiated by sending this list of updated
intermediate keys to the (first of the) new participant(s) to join.
The new participant(s) perform the key agreement protocol in exactly
the same fashion as done in the IKA upflow by generating their own
private key contributions, performing DH computations with them on the
intermediate keys and extending the intermediate key list with their
"own" intermediate key.

The last (new) participant in the extended list now will initiate the
downflow broadcast message consisting of *all* intermediate keys, thus
enabling every participant to compute the new shared group key and
reach a ready state.

Using the AKA for joins it is possible to add new participants either
one by one or multiple at the same time.  It is more efficient to add
multiple new participants at the same time than to add them
sequentially.

**Example:**

The following figure shows addition of a participant (:math:`p5`) --
initiated by :math:`p1` -- to the existing group of four participants.

.. digraph:: gka_aka_join
   :caption: Messages involved in an AKA join protocol flow for the
             addition of a single participant.

   layout=circo;
   size=2;
   ordering=out;
   node [style=filled, shape=circle];
   p5 [style=dashed];
   p1 -> p2 -> p3 -> p4 [style=invis];
   p1 -> p5 [arrowhead=empty, color=blue, label="u1'"];
   p5 -> {p1 p2 p3 p4} [label="d'"];

Message contents (Note: :math:`x_1` is the initiator's old private key
contribution, :math:`x_1'` is the new contribution):

* :math:`u1'`:

   * Participants: :math:`(p1,\; p2,\; p3,\; p4,\; p5)`
   * Intermediate keys: :math:`(x_4x_3x_2G,\; x_1'x_4x_3x_1G,\; x_1'x_4x_2x_1G,\; x_1'x_3x_2x_1G,\; x_1'x_1x_4x_3x_2G)`

* :math:`d'`:

   * Participants: :math:`(p1,\; p2,\; p3,\; p4,\; p5)`
   * Intermediate keys: :math:`(x_5x_4x_3x_2G,\; x_5x_1'x_4x_3x_1G,\; x_5x_1'x_4x_2x_1G,\; x_5x_1'x_3x_2x_1G,\; x_1'x_1x_4x_3x_2G)`

Again, after receiving these intermediate keys, every participant can
compute the same shared group key by multiplying "their" intermediate
key with their own private contribution:

.. math::

   x_1'x_1x_5x_4x_3x_2G = x_2x_5x_1'x_4x_3x_1G = x_3x_5x_1'x_4x_2x_1G = x_4x_5x_1'x_3x_2x_1G = x_5x_1'x_1x_4x_3x_2G


Member Exclusion
----------------

The AKA protocol flow for member exclusion is similar to -- but
simpler -- than member addition.  The initiator will update their
private key contribution (see :ref:`note_key_contributions`) in the
same manner as for the join above.  Then the participant(s) as well as
their intermediate key(s) are removed from the respective lists for
the participant(s) to be excluded.  Now the downflow broadcast message
can be sent directly without the need of a preceding upflow phase.
Thus, all remaining participants can compute the new shared group key
and reach a ready state.

Using the AKA for exclusion it is possible to remove participants
either one by one or multiple at the same time.  It is more efficient
to exclude multiple participants at the same time than sequentially.


Member Departure
----------------

Member departure is the "voluntary" parting of a participant rather
than an exclusion through another participant.  In effect it is the
same, with the only difference that the departing member indicates the
desire to leave, and a member exclusion AKA will be initiated upon
that by another participant (the owner).


Key Refresh
-----------

To avoid "key burn out" over long periods of key use, it is a good
idea to refresh the group key at suitable intervals (e. g. depending
on time, number of messages or volume encrypted with it).  A key
refresh is very simple, and can be initiated by *any* participant.
The initiating participant renews their own private key contribution
(see :ref:`note_key_contributions`), and broadcasts a downflow message
with all updated intermediate keys to all participants without the
need of a preceding upflow.  Thus, all participants can compute the
new shared group key and reach a ready state.

It is wise for participants to track the "age" of their own private
key contribution.  This mechanism can be used for achieving a
"rolling" group key refresh by always updating the oldest private key
contributions of participants.


.. _note_key_contributions:

Note on Updated Private Key Contributions
-----------------------------------------

When the private key contribution (for a join, exclusion or refresh)
is updated, the client must collect *all* the key contributions in a
list.  When performing computations to derive a new cardinal key, the
whole chain of one's own private key contributions needs to be used.
Using modular exponentiation from the common Diffie-Hellman approach,
these individual contributions can be condensed through multiplication
into a single value.  For the elliptic curve Diffie-Hellman a sequence
of subsequent scalar multiplications is required to be performed, as
operations do not possess the same associative properties, even though
commutativity is given.

In case this sequence may grow big over time, so that the overhead of
applying a long sequence of elliptic curve scalar multiplications can
become more significant.  In such cases, it may be worth to re-key the
whole session.


..
    Local Variables:
    mode: rst
    ispell-local-dictionary: "en_GB-ise"
    mode: flyspell
    End:
