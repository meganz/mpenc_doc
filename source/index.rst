========================================
Multi-Party Encrypted Messaging Protocol
========================================

This document is a technical overview and discussion of our work, a protocol
for secure group messaging. By *secure* we mean for the actual users i.e.
end-to-end security, as opposed to "secure" for irrelevant third parties.

Our work provides everything needed to run a messaging session between real
users on top of a real transport protocol. That is, we specify not just a key
exchange, but when and how to run these relative to transport-layer events; how
to achieve *liveness properties* such as reliability and consistency, that are
time-sensitive and lie outside of the send-receive logic that cryptography-only
protocols often restrict themselves to; and offer suggestions for displaying
accurate (i.e. secure) but not overwhelming information in user interfaces.

We aim towards a general-purpose unified protocol. In other words, we'd prefer
to avoid creating a completely new protocol merely to support automation, or
asynchronity, or a different transport protocol. This would add complexity to
the overall ecosystem of communications protocols. It is simply unnecessary if
the original protocol is designed well, as we have tried to do.

That aim is not complete -- our full protocol system, as currently implemented,
is suitable only for use with certain instant messaging protocols. However, we
have tried to separate out conceptually-independent concerns, and solve these
individually using minimal assumptions even if other components make extra
assumptions. This means that many components of our full system can be reused
in future protocol extensions, and we know exactly which components must be
replaced in order to lift the existing constraints on our full system.

Our main implementation is a JavaScript library; an initial version is awaiting
integration into the MEGA web chat platform. We also have a Python reference
prototype that is semi-functional but omits real cryptographic algorithms. Both
our design and implementation need external review. To that end, this document
contains high-level descriptions of each, and we have also published source
code and API documentation under the free open source software AGPL-3 license.

In chapters 1 and 2, we explore topics in secure group messaging in general,
then apply these to our work and justify our design choices. In chapter 3, we
present our full protocol system, including our chosen separations of concerns
and how we achieve various security properties. In chapter 4, we describe our
implementation, including our software architecture, along with paradigms we
employ in lower-level utilities. In chapter 5, we detail our crytographic
algorithms and our packet format. Finally in chapter 6, we outline future work,
both for the immediate short term as well as long-term research objectives.

.. only:: latex

    .. raw:: latex

        \newpage

    .. include:: copyright.rst

.. toctree::
   :maxdepth: 2
   :numbered:

   background
   topics
   protocol
   engineering
   crypto
   future-work
   references
