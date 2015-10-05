===========
Engineering
===========

In this chapter we describe the high-level architecture of the reference
implementation of our protocol system, and its core engineering concepts.

For a distributed system, the only *requirement* for it to work is that the
protocol is well-defined. However, protocols that support advanced features or
give strong guarantees are inherently more complex. Some people may argue that
the complexity is not worth the benefit, especially if it includes new ideas
untested by existing protocols. Our hope is that a high-level description of an
*implementation* will help reduce this over-estimation of the complexity.
Another hope is that it will make audits and code reviews easier and faster.

Public interface
================

We begin by observing that any implementation of any communications system must
(a) do IO with the network; and (b) do IO with the user. In our system, we
define the following interfaces for these purposes:

Session (interface)
  This represents a group messaging session that follows model in the previous
  chapter. It defines an interface for higher layers (e.g. the UI) to interact
  with, which consists of (a) one input method to attempt to send a message or
  change the session membership; (b) one output pipe for the user to receive
  session events or security warnings; and (c) various query methods, such as
  to get its current membership, the stream of messages accepted so far, etc.
  [#sess]_

GroupChannel (interface) [$]
  This represents a group transport channel. It defines an interface for higher
  layers (e.g. a Session) to interact with, which consists of (a) one input
  method to attempt to send a packet or change the channel membership; (b) one
  output pipe to receive channel events; and (c) various query methods. Actions
  may be ignored by the transport or satisfied exactly once, possibly after we
  receive further channel events not by us. The transport is supposed to send
  events to all members in the same order, but members must verify this.

| [$] This component is specific to instant or synchronous messaging; ones
  *not* marked with this may be re-used in an asynchronous messaging system.

Session represents a logical view from the point of view of one user; there is
no distinction between "not in the session" vs "in the session, and we are the
only member". By contrast, GroupChannel is our view of an external entity (the
channel), and the corresponding concepts for channel membership *are* distinct.

So to use our library, an external application must:

1. Implement GroupChannel for the chosen transport, e.g. XMPP.
2. Construct an instance of HybridSession (below), passing an instance of (1)
   to it along with other configuration options.
3. Hook the application UI into the API provided by Session. The transport
   layer may be ignored completely, since that is handled by our system.

The last step may involve a lot of work if the application UI is too tightly
coupled with the specifics of a particular protocol. But there is no way around
this; a secure messaging protocol deals with concepts that are fundamentally
different from insecure transport protocols, and we see this already by the
difference between session and channel membership. Hopefully whoever does this
work will architect their future software with greater foresight.

The remainder of this document contains a description of our implementation;
but knowledge beyond this point is not necessary merely to *use* our system.

Session architecture
====================

HybridSession [$] is our main (and currently only) Session implementation. It
contains several internal components:

- a GroupChannel, a transport client for communication with the network;
- a concurrency resolver, to gracefully prevent membership change conflicts;
- a component [*] that manages and runs membership operations and proposals;
- two components for the current and previous subsession; each contains:

  - a message encryptor/decryptor [*] for communicating with the session;
  - a transcript data structure to hold accepted messages in the correct order;
  - various liveness components to ensure reliability and consistency.

HybridSession itself handles the various transport integration cases; creates
and destroys subprocesses to run membership operations; and manages membership
changes that are initiated by the local user, that require more tracking such
as retries in the case of transport hiccups, etc.

The receive handler roughly runs as follows. For each incoming channel event:

1. if it is a channel membership change, then react to as part of transport
   integration [TODO: link];
2. else, if it is a membership operation packet:

   - if it is relevant to the concurrency resolver, pass it to that, which may
     cause an operation to start or finish (with success or failure);
   - if an operation is ongoing, pass it to the subprocess running that;
   - else reject the packet - it's not appropriate at this time.

3. else, try to verify-decrypt it as a message in the current subsession;

   - if the packet verifies but fails to be accepted into the transcript due
     to missing parents, put it in a try-accept queue *in this subsession*, to
     try this process again later (and similarly for the next case);

4. else, try to verify-decrypt it as a message in the previous subsession;
5. else, put it on a queue, to try this process again later, in case it was
   received out-of-order and depends on missing packets to decrypt.

The components that deal directly with cryptography are marked [*] above. These
may be improved independently from the others, and from HybridSession. We may
also replace the cryptographic primitives within each component - e.g. DH key
exchange, signature schemes, hash functions and symmetric ciphers - as
necessary, based on the recommendations of the wider community.

For more technical details, see our API documentation. [TODO: link].

Internal components
===================

ServerOrder [$]
  The concurrency resolver, used by HybridSession to enforce a consistent and
  context preserving total-ordering of membership operations. It tracks the
  results of older operations, whether we're currently in an operation, and
  decides how to accept/reject proposals for newer operations.

Greeter, Greeting (interface) [$]
  Greeting is a component representing a multi-packet operation, defined as an
  interface with a packet-based transport consisting of (a) one input method to
  receive data packets; (b) one output pipe to send out data packets; and (c)
  various query methods, such as to get a Future for the operation's result, a
  reference to any previous operation, the intended next membership, etc.
  Typically, this may be implemented as a state machine.

  Greeter is a factory component for new Greeting instances, defined as an
  interface used by HybridSession that consists of some limited codec methods
  for initial/final packets of a group key agreement. Implementations of these
  methods may reasonably depend on state, such as the result of any previous
  operation, data about operations proposed by the local user but not yet
  accepted by the group, or a reference to an ongoing Greeting.

SessionBase
  This is a partial Session implementation, for full implementations to build
  on top of or around (as HybridSession does). It enforces properties such as
  strong message ordering, reliability, and consistency, based on information
  from message parent references and using some of the components below.

  The component provides an interface with a packet-based transport consisting
  of (a) one input method to receive data packets; (b) one output pipe to send
  out data packets; and an interface with the UI consisting of (c) one output
  pipe for the user to receive notices; (d) various action methods for the user
  to call, such as sending messages and ending the session; and (e) various
  query methods similar to those found in Session.

  Unlike with Session(a), there is no attempt to simplify SessionBase(d) to
  make it "nice to use". The functionality is quite low-level and may change in
  the future; it is not meant for external clients of our system.

Everything from here on are components of SessionBase; HybridSession does not
directly interact with them.

MessageSecurity (interface)
  This defines an interface for the authentication and encryption of messages.
  The interface is flexible enough to allow implementations to generate new
  keys based on older keys, and to implement automatic deletion rules for some
  of those keys as they age further.

Transcript, MessageLog
  These are append-only data structures that hold messages in partial order.

  Transcript is a data structure that holds a causal-ordering of all messages,
  including non-content messages used for flow control, and other non-user
  concerns. It provides basic query methods, and graph traversal and recursive
  merge algorithms. (The latter is only for aiding future research topics.)

  MessageLog presents some linearisation of this for UX purposes, optionally
  aggregates multiple transcripts (from multiple subsessions in HybridSession),
  and filters out non-content messages whilst retaining relative ordering.

FlowControl
  This defines an interface that SessionBase consults on liveness issues, such
  as when to resend messages, how to handle duplicate messages, how to react to
  packets that have been buffered for too long, etc. The interface is designed
  to support using the same component across several SessionBase instances, in
  case one wishes to make decisions based on all of their states. The interface
  is private for the time being, since it is a little bit unstructured and may
  be changed later to fix this and other imperfections.

ConsistencyMonitor
  This is a component that tracks expected acknowledgements for abstract items,
  and issues warnings and/or tries to recover, if they are not received in a
  timely manner. It is used by SessionBase and (in the future) ServerOrder.

PresenceTracker
  This is a component that tracks and renews own and others' latest activity in
  a session, and issues warnings if these expire. This helps to detect drops by
  an unreliable transport or malicious attacker.

.. [#sess] We do not define a lower (transport) interface in Session because
    implementations or subtypes may require a *particular* transport, so they
    define what that is. For example, HybridSession requires a GroupChannel
    which makes it unsuitable for asynchronous messaging; but another subtype
    of Session might support that.

Utilities
=========

Our protocol system is built from components that act as independent processes,
that react to inputs and generate outputs similar to the actor model. We build
up a relatively simple framework for this intra-process IO, based on some
low-level utilities. We'll talk about these first.

Low-level
---------

For an input mechanism into a component that is decoupled from the source, we
simply use a function, since this exists in all major languages, and already
has the property that the callee doesn't know who the caller is.

For an output mechanism from a component that is decoupled from the target, we
use a synchronous publish-subscribe pattern. There are other options; the main
reason we choose this is that *how* we consume inputs (of a given type) changes
often. For example: each new message adds a requirement that we do some extra
things on future messages; in trial decryption, the set of possible options
changes; etc. Pub-sub is ideal for these issues: we can subscribe new consumers
when we need to, and define the behaviour of these, as well as when to cancel
the subscription, together in the source code.

By contrast, other intra-process IO paradigms (e.g. channels) are mostly built
around single consumers. Here, we'd have to collect all possible responses into
the consumer, then add explicit state to control the activation of specific
responses. This causes related concerns to be separated too much, and unrelated
concerns to be grouped together too much, and the mechanisms for this are less
standardised in libraries.

By "synchronous" we mean that the publisher executes subscriber callbacks in
its own thread. We understand the issues around this, but in our simple usage
it makes reasoning about execution order more predictable, and means that we
have no dependency on any specific external execution framework.

For long-running user-level operations, we use Futures, which is the standard
utility for this sort of asynchronous "function call"-like operation, that is
expected to return some sort of response. In our system, a common pattern is
for a Future's lifetime to include several IO rounds between components.

We chose to implement our own utilities for some of these things, to define
them in a more abstract style that is inspired from functional programming
languages. This allows us to write higher-order combinators, so that we can
express complex behaviours more concisely and generally.

Observable
  A pair of functions (publish, subscribe) and some mutable tracking state,
  used to produce and consume items. The producer creates an instance of this,
  keeps (publish) private and gives (subscribe) to potential consumers. In a
  language that supports polymorphic types, we would have the following type
  definitions, written in Scala-like pseudocode:

  .. code-block:: scala

    type Cancel             = () => Boolean
    type Subscribe[T, S]    = (T => S) => Cancel
    type Publish[T, S]      = T => List[S]

  ``T`` is the type of the communicated item, and ``S`` is an optional type
  (default ``Unit``) that callbacks may want to pass back to the producer, to
  signal some sort of "status". The return value of ``Cancel`` is whether the
  subscription was not already cancelled.

  Even if absent from the language, having an idea on what types *ought* to be
  helps us to write combinators, e.g. to make a complex subscribe function
  ("run A after event X but run B instead if event Y happens first and run A2
  if event X happens after that") or a complex cancel function ("cancel all in
  X and if all of them were already cancelled then also cancel all in Y").

EventContext
  A utility that supports efficient prefix-matched subscriptions, so consumers
  can specify a filter for the items they're interested in. The type signature
  of its public part is something like ``_Prefix_[T] => Subscribe[T, S]``,
  pretending for now that ``_Prefix_`` is a real type.

Timer
  Execute something in the future. Its type is simply ``Subscribe[Time, Unit]``
  so that it can be used with combinators. When integrating our library into an
  application, one can simply write an adapter that satisfies this interface,
  for whichever execution framework is used.

Future
  We only use these for user-level actions, so we don't need many combinators
  for them. Standard libraries are adequate for our use cases, e.g. Promise
  (JS) or defer.Deferred (Python).

We also have more complex utilities like Monitor, built on top of Observable
and its friends, used to implement liveness and freshness behaviours. For more
details, see the API documentation [TODO: link].

High-level
----------

We define two interfaces (*trait* or *typeclass* in some languages) as a common
pattern for our actor-like components to use. Each interface is essentially a
(function, subscribe-function) pair. The former is used for input into the
component, the latter for accepting output from it.

One interface is for interacting with a more "high level" component, e.g. a
user interface:

.. code-block:: scala

  trait ReceivingSender[SendInput, RecvOutput] {
    def onRecv : Subscribe[RecvOutput, Boolean] // i.e. (RecvOutput => Boolean) => (() => Boolean)
    def send   : SendInput => Boolean
  }

For example, when the UI wants to send some things to our session, it passes
this request to ``Session.send``. To display things received from the session,
it hooks into ``Session.onRecv``.

Another interface is for interacting with a more "low level" component, e.g. a
transport client:

.. code-block:: scala

  trait SendingReceiver[RecvInput, SendOutput] {
    def onSend : Subscribe[SendOutput, Boolean] // i.e. (SendOutput => Boolean) => (() => Boolean)
    def recv   : RecvInput => Boolean
  }

For example, when we want to tell a GKA session membership operation that we
received a packet for it, we call ``Greeting.recv``. To service its requests to
send out response packets, we hooks into ``Greeting.onSend``.

Here are some examples of our components that implement the above interfaces:

.. code-block:: scala

  trait Session         extends ReceivingSender[SessionAction, SessionNotice];
  trait GroupChannel    extends ReceivingSender[ChannelAction, ChannelNotice];
  trait Greeting        extends SendingReceiver[RawByteInput, RawByteOutput];
  class SessionBase     extends SendingReceiver[RawByteInput, RawByteOutput];

  type RawByteInput     = (SenderAddr, Array[Byte])
  type RawByteOutput    = (Set[RecipientAddr], Array[Byte])

These interfaces are also used privately too, to maintain a common style for
the code architecture. For example ``HybridSession`` contains an implementation
of ``SendingReceiver[ChannelNotice, ChannelAction]``, but this is not exposed
since it is just an implementation detail, and it is only meant to be linked
with the associated ``GroupChannel``.

We define ``S`` for ``Subscribe[T, S]`` as ``Boolean`` in these interfaces for
simplicity, meaning "the item was {accepted, rejected} by the consumer". This
allows us to detect errors - such as transport failures in sending messages, or
trial decryption failures in receiving packets - but in a loosely-coupled way
that discourages violation of the separation of layers. One reasonable
extension is to use a 3-value logic to represent {accept, try later, reject},
which helps both of the previous cases.

This concludes the overview of our reference implementation. All the code that
is not mentioned here, are straightforward applications of software engineering
principles or algorithm writing, as applied to our protocol design (previous
chapter) and software design (this chapter). For more details, see the API
documentation and/or source code.
