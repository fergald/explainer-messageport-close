# MessagePort close explainer

## Metadata

**Author**: [Fergal Daly](mailto:fergal@chromium.org)
**Status**: Draft

## What is this?

This is a proposal to add a `close` event
which fires when a `MessagePort`'s entangled port is closed
or becomes unreferenced
(e.g. it was owned by a process that has crashed).

### Problem to be addressed

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
Freeing resources prematurely would be a problem there.
So a timeout based approach does not work
for the general case.

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
without ever prematurely freeing resources.

#### Lifecycle events and WeakRefs

Relying entirely on GC
may result in an arbitrarily long delay.
Adding some communication of lifecycle state can help.
Having the other side communicate its lifecycle state,
e.g. by sending a message when the document is being destroyed
or navigated away from,
would allow earlier cleanup.
This could be done in a `pagehide` event handler
when the event's `persisted` field is `false`.
The cases where the other side
- crashes
- enters BFCache and is destroyed without being restored
- fails to communicate
are covered by GC.

### Concerns

In the [issue](https://github.com/whatwg/html/issues/1766) there are 2 main concerns expressed about any solutions.

#### Exposing that cross-origin navigation has occurred

By passing a `MessagePort` to a cross-origin document
and listening to the `close` event,
it is possible to find out when that document has been destroyed.
This can be done with without the cooperation of the document in question.

**This information is already available**
through the use of `WeakRef`s.
The concern here can only be about the *timing*
of this information.
There would not be any new capability granted
by allowing delivering this event *eventually*.

Right now a page that wanted this information
as close as possible to the destruction event
could poll the `WeakRef` frequently
while allocating engaging in activity
that triggers GC.
A demo of this is [here](https://message-port-weakref-forced-gc-learning.glitch.me/)
([source](https://glitch.com/edit/#!/message-port-weakref-forced-gc-learning?path=index.html%3A72%3A0)).

It's unclear that there would be any now capability granted
by delivering this event *promptly*
if the observer is willing to behave in a way
that triggers GC.

**Question**: Is there a specific problem or attack
that is enabled by delivering the event in a timely manner?

#### Exposing garbage collection

Since this event would need to fire
in reaction to a port becoming GCed,
the timing of the event reveals the occurrence of GC.

**Question**: Why is this a concern?
Polling `WeakRef`s already seems to give this ability.
This would potentially give it cross-origin
but that would only be the case
if the port was closed by becoming unreferenced.
The listener cannot know whether the port was
explicitly closed,
last reference was dropped
or if the owning document has been destroyed.
Is it simply that exposing GC
is something to be avoided when possible?

## Proposal

Add a `close` event that fires
when an entangled `MessagePort` is closed.
Given a pair of entangled ports, `portA` and `portB`,
if `portA` is closed,
the event is fired on `portB`.
This `close` event is used as follows:
```
portB.onclose = () => {
  console.log("portA has been closed.");
  // Free up resources associated with the port.
}
```

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
when that port stops being "entangled" with the other port
(as specified [here](https://html.spec.whatwg.org/multipage/web-messaging.html))

The implications are discussed for each case.

### `close()` is called

Firing the `close` event as soon as possible
after the entangled port's `close()` method is called seems appropriate.
This is an explicit action taken by holder of the object
and it reveals nothing about GC.

### Owning Document Crashes

In this case,
from an implementation point of view,
it is possible to fire the event
on entangled port immediately
(subject to inter-process delays).

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
it seems impossible to avoid exposing
the timing of GC or navigation
in some cases.
Given that it is already effectively exposed,
this accepts that and provides an ergonomic solution.

## Alternatives considered

### Alternative timings

If we only send the event
on an explicit `close()`
or when the document owning the object is destroyed
then we avoid exposing GC timing.
If we do this without also changing the entanglement timing
then `WeakRef` can still be used to observe GC timing.
Changing when entanglement ends is doable
but currently it ends as soon as possible in all cases.
This would add some complication.
It would also indefinitely extend to the entangled lifetime of some `MessagePorts`
(those that currently end via GC).

### Require action by the receive of the MessagePort

To avoid exposing the information
about navigations occurring in cross-origin frames,
the `close` event would not be sent
unless the owning document took some explicit action
after receiving the `MessagePort` via `postMessage`.
Suggested [here](https://github.com/whatwg/html/issues/1766#issuecomment-960328593)

Given that `WeakRef`s inevitably expose
the disentanglement of `MessagePort`s already,
it does not seem useful to try to hide that.

## TAG Security & Privacy Questionnaire Answers

* **Author:** murakinonoka@google.com
* **Questionnaire Source:** https://www.w3.org/TR/security-privacy-questionnaire/#questions

### Questions

* **What information might this feature expose to Web sites or other parties, and for what purposes is that exposure necessary?**
  * This feature exposes that cross-navigation’s destruction
   or garbage collection has occurred.
   By passing a MessagePort to a document and listening to a close event,
   it is possible to find out
   when the document has been destroyed
   or the port has been garbage collected.
   However, this information is already exposed
   by polling WeakRef frequently
   and the `close` event enables this information
   to be exposed more timely and reliably.
   In addition to that,
   the listener cannot know if the close event is caused
   by document destruction/explicit close/just general GC due to being unreferenced.
* **Do features  in your specification expose the minimum amount of information necessary to enable their intended uses?**
  * Yes.
   The `close` event carries no other information
   except that the port is now closed.
* **How do the features in your specification deal with personal information, personally-identifiable information (PII), or information derived from them?**
  * This proposal does not deal with PII.
* **How do the features in your specification deal with sensitive information?**
  * This proposal does not deal with sensitive information.
* **Do the features in your specification introduce new state for an origin that persists across browsing sessions?**
  * No.
* **Do the features in your specification expose information about the underlying platform to origins?**
  * No.
* **Does this specification allow an origin to send data to the underlying platform?**
  * No.
* **Do features in this specification enable access to device sensors?**
  * No.
* **Do features in this specification enable new script execution/loading mechanisms?**
  * No.
* **Do features in this specification allow an origin to access other devices?**
  * No.
* **Do features in this specification allow an origin some measure of control over a user agent’s native UI?**
  * No.
* **What temporary identifiers do the features in this specification create or expose to the web?**
  * None - no new identifiers are created or exposed.
* **How does this specification distinguish between behavior in first-party and third-party contexts?**
  * No.
* **How do the features in this specification work in the context of a browser’s Private Browsing or Incognito mode?**
  * No differently.
* **Does this specification have both "Security Considerations" and "Privacy Considerations" sections?**
  * Yes.
* **Do features in your specification enable origins to downgrade default security protections?**
  * No.
* **How does your feature handle non-"fully active" documents?**
  * If the document doesn't get destroyed on navigation
   and instead is kept around non-fully active/BFCached
   it will not fire.
   It will fire if the document is discarded while non-fully active.
* **What should this questionnaire have asked?**
  * Nothing to add.
  
