===========
Engineering
===========

API for external developers
===========================

- logical difference between session vs channel

etc..

Implementation
==============

High-level components
---------------------

HybridSession, MessageLog, Transcript/SessionBase, ServerOrder

Utilities and primitives
------------------------

- trial decryption + acceptance

- async utils, primitives
  - event objects
  - independence from a particular execution framework (e.g. twisted or whatever)

- SendingReceiver / ReceivingSender

- issues: sync vs async publishing (in an Observable/Promise)

UI considerations
=================

Reference msg-notes
