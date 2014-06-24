---
title: Messaging
sequence: 2
description: "Simple creation and usage of distributed queues and topics"
date: 2014-06-23
---

If you're coming from Immutant 1.x, you may notice that the artifact
has been renamed (`org.immutant/immutant-messaging` is now
`org.immutant/messaging`), and the API has changed a bit. We'll point
out the notable API changes as we go.

## The API

The messaging [API](#{api_doc_for_2x_version('LATEST')}) is backed by
[HornetQ], which is an implementation of [JMS]. JMS provides two
primary destination types: *queues* and *topics*. Queues represent
point-to-point destinations, and topics publish/subscribe.

To use a destination, we need to get a reference to one via the
[queue](#{api_doc_for_2x_version('LATEST', 'messaging', 'queue')}) or
[topic](#{api_doc_for_2x_version('LATEST', 'messaging', 'topic')})
functions, depending on the type required. This will create the
destination if it does not already exist. This is a bit different than
the 1.x API, which provided a single `start` function for this, and
determined the type of destination based on conventions around the
provided name. In 2.x, we've removed those naming conventions.

Once we have a reference to a destination, we can operate on it with
the following functions:

* [publish](#{api_doc_for_2x_version('LATEST', 'messaging', 'publish')}) -
  sends a message to the destination
* [receive](#{api_doc_for_2x_version('LATEST', 'messaging', 'receive')}) -
  receives a single message from the destination
* [listen](#{api_doc_for_2x_version('LATEST', 'messaging', 'listen')}) -
  registers a function to be called each time a message
  arrives at the destination

If the destination is a queue, we can do synchronous messaging
([request-response]):

* [respond](#{api_doc_for_2x_version('LATEST', 'messaging', 'respond')}) -
  registers a function that receives each request, and the
  returned value will be sent back to the requester
* [request](#{api_doc_for_2x_version('LATEST', 'messaging', 'request')}) -
  sends a message to the responder

Finally, to deregister listeners, responders, and destinations, we
provide a single
[stop](#{api_doc_for_2x_version('LATEST','messaging', 'stop')})
function. This is another difference from 1.x -
the `unlisten` and `stop` functions have been collapsed to `stop`.

### Some Examples

You should follow the instructions in the [installation] tutorial to
set up a project using Immutant 2.x, and in addition to
`org.immutant/messaging` add `[cheshire "5.3.1"]` to the project
dependencies (we'll be encoding some messages as JSON in our examples
below, so we'll go ahead and add
[cheshire](https://github.com/dakrone/cheshire) while we're at it).
Then, fire up a REPL, and require the `immutant.messaging` namespace
to follow along:

<pre class="syntax clojure">(require '[immutant.messaging :refer :all])</pre>

First, let's create a queue:

<pre class="syntax clojure">(queue "my-queue")</pre>

That will create the queue in the HornetQ broker for us. We'll need a
reference to that queue to operate on it. Let's go ahead and store
that reference in a var:

<pre class="syntax clojure">(def q (queue "my-queue"))</pre>

We can call `queue` any number of times - if the queue already exists,
we're just grabbing a reference to it.

Now, let's register a listener on our queue. Let's just print every
message we get:

<pre class="syntax clojure">(def listener (listen q println))</pre>

We can publish to that queue, and see that the listener gets called:

<pre class="syntax clojure">(publish q {:hi :there})</pre>

You'll notice that we're publishing a map there - we can publish
pretty much any data structure as a message. By default, that message
will be encoded using [edn]. We also support other encodings, namely:
`:fressian`, `:json`, and `:none`. We can choose a different encoding
by passing an :encoding option to `publish`:

<pre class="syntax clojure">(publish q {:hi :there} :encoding :json)</pre>

If you want to use `:fressian` or `:json`, you'll need to add
`org.clojure/data.fressian` or `cheshire` to your dependencies to
enable them, respectively.

We passed our options to `publish` as keyword arguments, but they can
also be passed as a map:

<pre class="syntax clojure">(publish q {:hi :there} {:encoding :json})</pre>

This holds true for any of the messaging functions that take options.

We're also passing the destination reference to `publish` instead of the
destination name. That's a departure from 1.x, where you could just pass the
destination name. Since we no longer have conventions about how queues and
topics should be named, we can no longer determine the type of the
destination from the name alone.

We can deregister the listener by either passing it to `stop` or
calling `.close` on it:

<pre class="syntax clojure">(stop listener)
;; identical to
(.close listener)</pre>

Now let's take a look at synchronous messaging. Let's create a new
queue for this (you'll want to use a dedicated queue for each
responder) and register a responder that just increments the request:

<pre class="syntax clojure">(def sync-q (queue "sync"))

(def responder (respond sync-q inc))</pre>

Then, we make a request, which returns a [Future] that we can
dereference:

<pre class="syntax clojure">@(request sync-q 1)</pre>

The responder is just a fancy listener, and can be deregistered the
same way as a listener.

## More to come

That was just a brief introduction to the messaging API. There are
features we've yet to cover (durable topic subscriptions,
connection/session sharing, transactional sessions, remote
connections)...

[HornetQ]: http://hornetq.jboss.org/
[JMS]: https://en.wikipedia.org/wiki/Java_Message_Service
[installation]: /tutorials/installation/
[request-response]: https://en.wikipedia.org/wiki/Request-response
[Future]: http://docs.oracle.com/javase/7/docs/api/java/util/concurrent/Future.html
[edn]: https://github.com/edn-format/edn
