===========
Future work
===========

Here, we discuss tasks for the future, adding functionality and building on our
existing security properties.

Immediate
=========

Missing properties
------------------

Some of the security mechanisms that we have described, have not actually been
implemented yet. We list them below. We have delayed them because we consider
them to be less-critical security properties, and wanted to focus on making our
implementation work reliably enough to start testing. However, we recognise
their importance at providing end-to-end guarantees; none of them are expected
to be hard to achieve, and solution outlines have already been sketched out.

Error and abort messages
  We should add the ability to abort a membership change, either manually or
  automatically after a timeout. Currently if someone disconnects, others will
  be left waiting until they themselves leave the channel. This must be done
  via an explicit "abort" packet that gets handled by the concurrency resolver,
  to prevent races between members.

  For liveness properties, we already have timeouts that emit local security
  warnings if good conditions are not reached after a certain time. However,
  user experience will benefit if we have explicit error messages that inform
  other members of conditions such as "inconsistent history", that *fail-fast*
  more quickly than our generous default timeouts.

Server-order consistency checks
  We need to implement consistency checks for accepted operations, since we use
  the server's packet ordering. This is similar to our consistency checks for
  messages: expect everyone to send authenticated acks to confirm their view of
  all previous operations. Further, the parent reference contained within the
  initial packet must also be authenticated by the initiator.

  Together, these prevent the server from re-ordering operations *completely*
  arbitrarily. However, given a set of concurrent operations with the same
  authenticated parent, it is still able to choose which one is accepted, by
  broadcasting that one first. Given other constraints, we think this security
  tradeoff is acceptable. [#rtry]_

Flow control
  We need recovery (automatic resends) for dropped messages, and heartbeats to
  verify in-session freshness. (We already have security warnings for messages
  received out-of-order.) This is already implemented in a non-production-ready
  Python prototype, with random integration tests, and only needs to be ported.

.. _publish-sess-sig-keys:

Publish signature keys
  We need to make signature-key publishing logic work properly, so that we have
  deniable authentication for message content.

  Roughly, once a member makes a subsession shutdown request (``FIN``), they
  may publish their signature key after everyone acks this request. This is
  safe (i.e. an attacker cannot reuse the key to forge messages), if we enforce
  that one may not author messages *after* a ``FIN``, i.e. all receivers must
  refuse to accept such messages. However, this simple approach destroys our
  ability to authenticate our own acks of others' messages (e.g. *their*
  ``FIN``) after we send our own ``FIN``. So we'll need something a bit more
  complex, and we haven't worked out the details yet.

  If others' acks to our ``FIN`` are blocked, then we will never be sure that
  it's safe to publish our signature key. This likely can't be defended under
  this type of scheme, since confidential authenticity isn't meaningful without
  authenticity (it would be "confidential nothing"); the equivalent attack also
  applies to OTR. To defend against this, we would need a session establishment
  protocol that is itself deniable, and then we don't need to mess around with
  publishing keys; see "Better membership change protocol" below.

.. [#rtry] Users may manually retry rejected operations as many times as they
   want, and it would be extremely suspicious if it is rejected too often. Note
   that automatic retries are a security risk.

Next steps
==========

Security improvements
---------------------

Messaging ratchet for intra-subsession forward secrecy
  We already have forward secrecy for old subsessions, but this is important
  for long-running subsessions and later when we do asynchronous messaging.

  One simple scheme is to deterministically split the key into :math:`n` keys,
  one for each sender. Then, each key can be used to seed a hash-chain ratchet
  for its associated sender. Once all readers have decrypted a packet and
  deleted the key, the forward secrecy of messages encrypted with that key and
  previous ones is ensured. However, since this scheme does not distribute
  entropy between members, there is no chance to recover from a memory leak and
  try to regain secrecy for future messages.

Better membership change protocol
  Use a constant-round group key exchange such as that from [np1sec]_, or even
  pairwise [Triple-DH]_ between all group members which extends better to
  asynchronous messaging. In both cases, we get deniability for free without
  having to publish signature keys for messages.

Use peer-to-peer or anonymous transport for non-GKA messages
  The only part of our system that requires a linear ordering is the membership
  operation packets. So it is possible to move message packets into a transport
  that gives us better properties (e.g. against metadata analysis) than XMPP.

More functionality
------------------

Large messages and file transfer
  Our current padding scheme limits messages to roughly :math:`2^{16}` bytes,
  to keep it under our XMPP server maximum stanza size. This may be extended to
  arbitrary sizes: pad to the next-power-of-2-multiple of the maximum size, and
  split this into MTU-sized packets. (This is just a functionality improvement;
  these padding schemes have not been researched from an adversarial model.)
  After this, it is a straightforward engineering task to allow file transfers.

Better multi-device support
  A group messaging system can already be used for multiple devices in a very
  basic way -- simply have each device join the session as a separate member.
  This is desireable for security reasons, since it means we can avoid sharing
  ephemeral keys. It's unclear whether devices should share identity keys, or
  use different identity keys and have the PKI layer bind them to say "we are
  the same identity", but this decision doesn't affect our messaging system.

  Beyond this, we can add some things both in the messaging layer and in the UI
  layer to make the experience smoother for users.

  - The users view should show one entry per user, not per device;
  - Not-fully-acked warnings may be tweaked to only require one ack from every
    *user* rather than every device. However, a warning should probably fired
    eventually even if all devices don't ack it, just later than in the single
    device case; it is still a critical security error if different devices get
    *different* content. Similar logic may be applied to heartbeats;
  - There are extra corner cases in the browser case, where the user may open
    several tabs (each acting as a separate device), with crashes and page
    reloads causing churn that might reveal implementation bugs.

  (We are already doing some of the above.)

Sync old session history across devices
  It is unnecessary to reuse security credentials (e.g. shared group keys or
  session keys) that are linked to others -- we already decrypted the packets
  and don't need to do this again. Futher, credentials in modern protocols are
  supposed to be ephemeral, and this is a vital part of their security. If we
  retain such credentials, we may put others at risk or leave forensic traces
  of our own activities.

  Therefore, our sync mechanism must not directly reuse ciphertext from our
  messaging protocol, since it forces us to store these credentials. It is much
  better to re-encrypt the plaintext under our own keys, unlinked to anyone
  else. That is, *at the very least*, this feature must be a separate protocol;
  the security model here is *private storage* for oneself, and *not* private
  communications. Finally, even following this requirement, long-term storage
  of encrypted data directly counteracts forward secrecy, so the user must be
  made aware of this before such a feature is enabled.

Research
========

Here are some research topics for the future for which we have no concrete
solution proposals, though we do have some vague suggestions.

Several of these relate to "no-compromise" asynchronous messaging, i.e. with
causal ordering, no breaking of symmetry between members, no requirement of
temporary synchronity or total ordering, no accept-reject mechanisms, and no
dependency on external infrastructure.

Merging under partial visibility
  As mentioned earlier, our membership operations are in a total order because
  nobody has defined how to merge two group key agreements. This problem has a
  well-defined solution for pairwise key agreements, but only if everyone can
  see all history, or if only member inclusions are allowed (or generally, if
  the operations to be merged have no inverse). If we have partial visibility
  (i.e. members can't see events from before they join) *and* we want to
  support both member inclusion and exclusion, the solution is unknown.

Session rejoin semantics
  As part of solving the above point, we need to decide what parent references
  mean exactly in the context of rejoining a session. Existing members' parent
  references to older messages won't make sense to us since we can't see them;
  symmetrically, we might want to reference the last messages we saw before
  previously leaving the session, but these references might not make sense to
  some of the existing members, i.e. those not present when we parted.

Possible hybrid solution
  One possible solution is to allow causally-ordered member inclusion, but
  require that everyone acknowledge a member exclusion before it is considered
  complete. Then our partial visibility problem disappears; new members don't
  have to worry about how to merge in excludes that happened before they joined
  -- their inviter will have already taken this into account. This is probably
  the least non-zero "compromise" solution, but the agreement mechanism might
  itself be very complex.

Save and load current session
  This is vital for asynchronous messaging, and would be a straightforward but
  significant engineering effort on top of our existing implementation.

  One optimisation to be made after the basic ability is complete, is to prune
  older messages from our transcript and message-log data structures. This must
  be thought through carefully, since we need a limited set of history in order
  to perform ratcheting, check the full-ack status of messages and freshness of
  other members, and merge concurrent membership operations.

Membership change *policy* protocol
  This ought to be layered on top of a membership change *mechanism* protocol.
  When reasoning about security, naturally one considers who is allowed to do
  what. But authorization is a separate issue from *how to execute membership
  changes*. We should solve the latter first, assuming that all members are
  allowed to make any change (in many cases this is exactly what is desired),
  *then* think about how to construct a secure mechanism to restrict these
  operations based on some user-defined policy. This is the same reason why we
  generally perform authentication before, and separately from, authorization.
