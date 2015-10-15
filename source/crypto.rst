============
Cryptography
============

Here, we document the specifics of how we use and implement cryptography in our
protocol, and describe the information contained in our packets.

This is *not* an overview of our full protocol system. Beyond processing and
generating packets, other algorithms and data structures must be implemented to
ensure a coherent high-level understanding of the state of the session, and the
ability to actively react to wider concerns such as liveness and consistency of
session state. These latter topics are covered in :doc:`protocol`.

Packet overview
===============

Every packet in our protocol is a sequence of records, similar to HTTP headers.
Each record has a type and a value, and some records may appear multiple times.

There are two packet types - (a) packets part of a membership change operation,
that we sometimes also call "greeting packet" or "greeting message"; and (b)
packets that represent a logical message written by members of the session.

Both packet types include the following common records, that occur either at
or near the start of the sequence, in order:

- ``MESSAGE_SIGNATURE`` - This is a cryptographic signature for the rest of the
  packet. This is calculated differently for greeting and data packets, defined
  in the respective sections below.
- ``PROTOCOL_VERSION`` - This indicates the protocol version being used, as a
  16-bit unsigned integer. Our current version is ``0x01``.
- ``MESSAGE_TYPE`` - Indicates the packet type, ``GREETING`` or ``DATA``.

Membership changes
==================

Our membership change protocol consists of two subprotocols, one to derive an
ephemeral encryption key and one to derive ephemeral signature keys.

More modern protocols with better security properties exist, but ours is a
simple approach using elementary cryptography only, which should be easier to
understand and review. It will be improved later, but for now is adequate for
our stated security goals, and was quick to implement.

The subprotocols are described in their own chapters; please read them first.
We introduce some terminology there, that is needed to understand the rest of
this document.

.. toctree::
   :maxdepth: 2
   :numbered:

   crypto_gka
   crypto_aske

There are four operation types: *establish session*, *include member*, *exclude
member* and *refresh group key*. Refresh may be thought of as a "no-op" member
change and that is how our overall protocol system treats it. Each operation
consists of the following stages:

Protocol upflow (optional)
  The protocol upflow consists of a series of directed messages; one user sends
  it to exactly one recipient, i.e. the next member in the ``MEMBER`` list of
  records. It is used to "collect" information from one member to the next. In
  a transport that *only* offers broadcast, the other recipients simply ignore
  packets not meant for them (i.e. the ``DEST`` record is not them).

  Every upflow message includes the records marked [1] from the summary below.
  The first upflow message also includes records marked [2], which contains
  metadata about the operation, used by the concurrency resolver.

  This stage is not present for exclude or refresh operations.

Protocol downflow
  The protocol downflow consists of a series of broadcast messages, sent to all
  recipients in the group. It is used to "distribute" information contributed
  by all participants (or derived therefrom) to all participants at once.

  Every downflow message includes the records marked [3] from the summary
  below. The first downflow message also includes records marked [1], which
  contains information from everyone, collected during the upflow. If the
  operation has no upflow (e.g. exclude and refresh), the first downflow
  message also includes records marked [2].

A greeting packet contains the following records, in order:

* ``MESSAGE_SIGNATURE``, ``PROTOCOL_VERSION``, ``MESSAGE_TYPE``
* ``GREET_TYPE`` What operation the packet is part of, as well as which stage
  of that operation the packet belongs to. [#gtyp]_
* ``SOURCE`` User ID of the packet originator.
* ``DEST`` (optional) User ID of the packet destination; if omitted, it means
  this is a broadcast (i.e. downflow) packet.
* ``MEMBER`` (multiple) Participating member user IDs. This record appears *n*
  times, one for each member participating in this operation, or in other words
  the set of members in the subsession that would be created on its success.
  This also defines the orders of participants in the upflow sequence.
* [1] ``INT_KEY`` (multiple) Intermediate DH values for the GKA.
* [1] ``NONCE`` (multiple) Nonces of each member for the ASKE.
* [1] ``PUB_KEY`` (multiple) Ephemeral public session signing key of a member.
* [2] ``PREV_PF`` Packet ID of the final packet of the previously completed
  operation, or a random string if this is the first operation.
* [2] ``CHAIN_HASH`` Chain hash corresponding to that packet, as calculated by
  the initiator of this operation. This allows joining members to calculate
  subsequent hashes of their own, to compare with others for consistency.
* [2] ``LATEST_PM`` (multiple) References to the latest (logical, i.e. data)
  messages from the previous subsession that the initiator of this operation
  had seen, from when they initiated the operation.
* [3] ``SESSION_SIGNATURE``: Authenticated :ref:`confirmation signature
  <aske-session-sig>` for the ASKE, signed with the long-term identity key of
  the packet author.

1. Only for upflow messages and the first downflow message. During the upflow,
   this gets filled to a maximum of *n* occurences.
2. Only for the initial packet of any operation. These fields are ignored by
   the greeting protocol; instead they are used by the concurrency resolver.
   See :ref:`transport-integration` and [msa-5h0]_ for details.
3. Only for downflow messages, to confirm the ASKE keys.

``MESSAGE_SIGNATURE`` is made by the ephemeral signing key and signs the byte
sequence ``$magic_number || all_subsequent_records``. ``$magic_number`` is a
fixed string to prevent the signature being injected elsewhere; for greeting
packets in this version, it is the byte sequence ``greetmsgsig``.

Even though this ephemeral key is not authenticated against any identity keys
at the start of the operation, it is so by the end via ``SESSION_SIGNATURE``.
It is important not to allow ongoing operations to affect anything else in the
overall session, until this authentication has taken place; our atomic design
satisfies this security requirement.

Likewise, members may mess with the structure of packets as the operation runs,
such as re-ordering the ``MEMBER`` list of records. However, there should be no
security risk; inconsistent results will be detected via ``SESSION_SIGNATURE``
and cause a failure of the overall operation.

.. [#gtyp] The exact details may be viewed in the source code. The current
    values are vestiges from previous iterations of the protocol, where we did
    things differently and with more variation. A better approach would be to
    infer this from the other records that are already part of the message, and
    eliminate the redundant information, which may be set to incorrect values
    by malicious participants. This will be done in a later iteration.

Messages
========

Once a subsession has been set-up after the completion of a membership change,
members may exchange messages confidentially (using the shared group key) and
authentically (using the ephemeral signing key).

A data packet contains the following records, in order:

* ``SIDKEY_HINT`` A one byte hint, that securely gives the recipient an aid in
  efficiently selecting the decryption key for this message.
* ``MESSAGE_SIGNATURE``, ``PROTOCOL_VERSION``, ``MESSAGE_TYPE``
* ``MESSAGE_IV`` Initialisation vector for the symmetric block cipher. The
  cipher we choose is malleable, to give better deniability when we publish
  ephemeral signature keys, similar to OTR [OTR-spec]_.
* ``MESSAGE_PAYLOAD``. Ciphertext payload. The plaintext is itself a sequence
  of records, as follows:

  * ``MESSAGE_PARENT`` (multiple) References to the latest (logical, i.e. data)
    messages that the author had seen, when they wrote the message.
  * ``MESSAGE_BODY`` UTF-8 encoded payload, the message content as written by
    the human author.

``MESSAGE_SIGNATURE`` is made by the ephemeral signing key and signs the byte
sequence ``$magic_number || H( sid || group_key ) || all_subsequent_records``.
``$magic_number`` is a fixed string to prevent the signature being injected
elsewhere; for data packets in this protocol version, it is the byte sequence
``datamsgsig``. Since a single ephemeral signing key may be used across several
subsessions, the hash value ensures that each ciphertext is verifiably bound to
a specific subsession. [#psig]_

**Padding**. We use an opportunistic padding scheme that obfuscates the lower
bits of the length of the message. This has not been analysed under an
aggressive threat model, but is very simple and cannot possibly be harmful. The
scheme is as follows:

- Define a baseline size ``size_bl``. We have chosen 128 bytes for this, to
  accommodate the majority (e.g. see [08MCIC]_) of messages in chats, while
  still remaining a multiple of our chosen cipher's block size of 16 bytes.
- The value of ``MESSAGE_BODY`` is prepended by a 16-bit unsigned integer (in
  big-endian encoding) indicating its size.
- Further zero bytes are appended up to ``size_bl`` bytes.
- If the payload is already larger than this, then we instead pad up to the
  next power-of-two multiple of ``size_bl`` (e.g. ``2 * size_bl``, ``4 *
  size_bl``, ``8 * size_bl``, etc) that can contain the payload.

**Trial decryption**. In general, at some points in time, it may be possible to
receive a message from several different sessions. The ``SIDKEY_HINT`` record
helps to efficiently determine the correct decryption key. We calculate it as
``H( sid || group_key )[0]``. ``sid`` is the session ID (of the subsession) as
determined from the ASKE, and ``group_key`` is its encryption key.

Only the first byte of the hash value is used, so it is theoretically possible
that values may collide and subsequent trials may be required to determine the
correct keys to use. However, this should give a performance improvement in the
majority of cases without sacrificing security.

When we implement forward-secrecy ratchets, or when we move to anonymous
transports that don't provide hints on the authorship of packets (as XMPP does
with the ``from`` attribute of stanzas), this technique may be adapted further
to select between the multiple decryption keys or multiple authentication keys
(respectively) that are available to verify-decrypt packets.

.. [#psig] When/if we come to publish ephemeral signature keys, we will also
    have to publish all ``H( sid || group_key )`` values that were used by the
    key, to ensure that a forger can generate valid packets without knowing the
    group encryption keys.

Encoding and primitives
=======================

To encode our records, we use TLV (type, length, value) strings, similar to the
OTR protocol. Messages are prepended with the fixed string ``?mpENC:`` followed
by a base-64 encoded string of all concatenated TLV records, followed by ``.``.
Type and length in the TLV records are each two octets in big-endian encoding,
whereas the value is exactly as many octets as indicated by the length.

This encoding may change at a later date.

The cryptographic primitives that we use are:

- Identity signature keys: ed25519_
- Session exchange keys: x25519_
- Session signature keys: ed25519
- Message encryption: AES128_ in `CTR mode`_
- Message references, packet references, trial decryption hints: SHA256_
- Key derivation: `HKDF`_-SHA256

These have generally been chosen to match a general security level of 128 bit
of entropy on a symmetric block cipher. This is roughly equivalent to 256 bit
key strength on elliptic curve public-key ciphers and 3072 bit key strength on
traditional discrete logarithm problem based public-key ciphers (e.g. DSA).

We may switch to a more modern block cipher at a future date, that is more
easily implementable and verifiable in constant time.

.. _ed25519: http://ed25519.cr.yp.to/
.. _x25519: http://cr.yp.to/ecdh.html
.. _AES128: https://en.wikipedia.org/wiki/Advanced_Encryption_Standard
.. _CTR mode: https://en.wikipedia.org/wiki/Block_cipher_mode_of_operation#Counter_.28CTR.29
.. _SHA256: https://en.wikipedia.org/wiki/SHA-2
.. _HKDF: https://tools.ietf.org/html/rfc5869
