===============
Progress report
===============

Security
========

We do not attempt to provide conf. of {existence, membership, metadata}, under
any attack scenario.

We attempt to provide "conf. of message content", "auth. of session
membership", "auth. of message content", and all listed in "higher-level",
under most typical attacks:

- under passive comms attack by outsider or insider:

  - retain all security properties for all members

- under active comms attack by outsider or insider:

  - retain all security properties for all members
  - (except for confidentiality vs insider attacker)

- under long-term leak of target T by outsider:

  - XXX TODO

- under long-term leak of target T by insider:

  - XXX TODO

- under all-memory compromise of target T by outsider or insider:

  - no guarantees

Re: false claims/omissions (by an attacker able to authenticate as a target) -
in our system the attacker is able to:

- make false non-receipt claims (not referring to seen messages)
- make false receipt claims (referring to unseen messages)

We don't specifically defend against this attack, since we believe that there
is no benefit to be gained by it. There is some discussion on how to prevent
this, though [link to msg-notes], if anyone wants to explore it in the future.

Functionality
=============

All of the above security properties, plus:

- Robust handling of disconnects and transport hiccups [*]
- Multi-device support
- "towards" async messaging

Mechanism
=========

- GKA to generate group enc key / eph sig keys (integrate previous paper)
- Extra system to prevent races (ServerOrder, link to msg-notes)
- Parent pointers, Message encryption
- Timeout warnings for "liveness" properties

Modularity
----------

- separation of higher properties, from simple properties

  - consistency/reliability/freshness no longer a concern of the GKA

- general interfaces for crypto components, e.g. GKA, ratchet
- independence from crypto primitives
