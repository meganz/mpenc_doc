===========
Future work
===========

Immediate
=========

- Use a real ratchet

  - most basic one is to split secret group key into n sender keys
    deterministically, then hash these keys as we receive/send messages out
    from that author

- Get signature-key publishing logic working properly for deniable auth.
- Consistency checks for server-order

- Better group key exchange
- Better flow control (resends)

- TODO: Multi-device support... what issues here?
- UI integration [TODO: refer to previous chapter, corresponding section]

Long term
=========

Engineering
-----------

- Consistency between python and javascript
- Translate into type-safe language (e.g. rust)
- Release async utils as standalone library

Functionality
-------------

- Shared chat history
- Move non-GKA messages into p2p/anon channel
- Asynchronous messaging

Comparison
==========

- Well-defined behaviour in case of concurrency
- free software
