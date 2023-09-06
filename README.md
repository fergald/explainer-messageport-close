# MessagePort close explainer

## Metadata

**Author**: [Fergal Daly](mailto:fergal@chromium.org)
**Status**: Draft

## What is this?

This is a proposal to add a `close` event
which fires when a `MessagePort`'s entangled port is closed
or becomes unreferenced
(e.g. it was owned by a process that has crashed).

## Background

[This issue](https://github.com/whatwg/html/issues/1766) has a long discussion of the problem and potential solutions.

### MessagePorts

The [`MessageChannel` API](https://developer.mozilla.org/en-US/docs/Web/API/MessageChannel) provides access to a pair of "entangled" ports.
[`MessagePort`](https://developer.mozilla.org/en-US/docs/Web/API/MessagePort)s.
A `MessagePort` can send/receive messages with its entangled port.
They can be passed to other frames,
including cross-origin frames.
Either of both ports in an entangled pair
can end up owned by documents that have no knowledge of each-other.

### Resource management is difficult

It may be that one holder of a `MessagePort` is holding resources
that are associated with it.
These resources should be freed
when the port is no longer useful.
There is no explicit signal available
as to when that has happened.

### Existing strategies for detecting when the port has closed

#### Free after timeout

The [spec](https://html.spec.whatwg.org/multipage/web-messaging.html#broadcasting-to-many-ports) includes this advice

> Broadcasting to many ports is in principle relatively simple: keep an array of `MessagePort` objects to send messages to, and iterate through the array to send a message. However, this has one rather unfortunate effect: it prevents the ports from being garbage collected, even if the other side has gone away. To avoid this problem, implement a simple protocol whereby the other side acknowledges it still exists. If it doesn't do so after a certain amount of time, assume it's gone, close the `MessagePort`` object, and let it be garbage collected.

Waiting for some time and assuming that the other side is gone
may be OK for some uses
but there are other uses where a port may have a long lifetime
or a lifetime/usage that is dependent on user actions.

#### WeakRefs

If the `MessagePort` is held only in a [`WeakRef`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WeakRef),
when the other side goes away,
drops the reference,
calls `close()`
or otherwise causes the connection between the ports to end,
the local `MessagePort` object becomes eligible for garbage collection (**GC**).
So, periodically checking the `WeakRef` and
freeing resources only if it `deref()`s to `null`
will avoid resource issues
without ever incorrectly freeing resources.

#### Lifecycle events

Relying entirely on GC
may result in an arbitrarily long delay.
One option is to have the other side
communicate its lifecycle state,
e.g. by sending a message when the document is being destroyed
or navigated away from.
This could be done in a `pagehide` event handler.
This cannot cover the case where the other side crashes
or enters BFCache and is destroyed without being restored.

### Concerns

In the [issue](https://github.com/whatwg/html/issues/1766) there are 2 main concerns expressed about any solutions.

#### Exposing that cross-origin navigation has occurred

By passing a `MessagePort` to a cross-origin document
and listening to the `close` event,
it is possible to find out when that document has been destroyed.
This can be done with without the cooperation of the document in question.

Given that this information is already available
through the use of `WeakRef`s,
any concern here can only be about the timing
of this information.
Right now a page that wanted this information
as close as possible to the destruction event
could poll the `WeakRef` frequently
while allocating and freeing large amounts of memory
to force GC.

There would be any new capability granted
by allowing delivering this event *eventually*.
It's unclear that there would be any now capability granted
by delivering this event *promptly*
if the observer can actively cause GC.

**Question**: Is there a specific problem or attack
that is behind this concern
or is it just the normal cross-origin isolation?

#### Exposing garbage collection

Since this event would need to fire
in reaction to a port becoming GCed,
the timing of the event reveals the occurrence of GC.

**Question**: Why is this a concern?
Polling `WeakRef`s already seems to give this ability.
This would potentially give it cross-origin
but that would only be the case
if the port was closed by become dereferenced.
The listener cannot know whether the port was
explicitly closed,
dereferenced
or if the owning document has been destroyed.
Is it simply that exposing GC
is something to be avoided when possible?

## Problem to be solved

There is no timely and reliable signal for when
a `MessagePort` has become unusable.
This makes it difficult to free resources
associated with ports.
The best that can be achieved is
timeliness in many cases
by reacting to lifecycle state changes
and reliability in the remaining cases
by depending on GC via `WeakRef`s.

By using all of these,
a comprehensive solution is actually possible
but it is unergonomic and
it does not seem to be well known.

## Proposal

Add a `close` event that fires
when an entangled `MessagePort` is closed.
Given a pair of entangled ports, `portA` and `portB`,
if `portA` is closed,
the event is fired on `portB`.

This can be done in such a way
that it exposes no information that isn't already exposed
but provides much better ergonomics
and gives us an opportunity to explicitly choose better timing.

There are several different scenarios to cover.
In each case,
the fact that the port has been closed
will inevitably become known to the holder of the entangled port
if they implement the strategies above.
So all that is really to be decided
is the timing of the event firing.

Implementation-wise it seems simplest
to fire the `close` event on a port
when that port stops being entangled with the other port.
The following cases follow that pattern.

### `close()` is called

Firing the `close` event as soon as possible
after the entangled port's `close()` method is called seems appropriate.
This is an explicit action taken by holder of the object
and it reveals nothing about GC.

### Owning Document Crashes

In this case,
from an implementation point of view,
it is possible to fire the event
on entangled port immediately.

### Owning Document is Destroyed

If the document is destroyed due to something other than a crash,
e.g. navigation or iframe deletion,
this may result in the `MessagePort` object becoming unreferenced
and then GCed some time in the future (see below)
or if this was the only document
in the JS execution context it may result in prompt GC.
See the next section.

### Port is Garbage Collected

At the point that the port is GCed,
we can fire the `close` event on the entangled port.

## Discussion

The choices above do not always avoid the concerns listed above
however it does not seem to be possible to avoid them fully.
In particular,
without e.g. adding random delays,
it seems impossible to avoid exposing
the timing of GC or navigation
in some cases.
Given that it is already effectively exposed,
this accepts that and provides an ergonomic solution.

## Alternatives considered

TODO(fergald): extract the other proposals from the issue.
I believe they are all trying to avoid exposing information
that is already exposed.
