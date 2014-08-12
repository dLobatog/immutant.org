---
title: Web
sequence: 1.5
description: "Deploying web apps built with popular Clojure libraries"
date: 2014-06-23
---

The `org.immutant/web` library changed quite a bit from Immutant 1.x
to 2.x, both its API and its foundation: the [Undertow] web server.
Among other things, this resulted in
[much better performance](https://github.com/ptaoussanis/clojure-web-server-benchmarks)
(~35% more throughput than v1.1.1) and built-in support for
websockets.

## The namespaces

The primary namespace, [immutant.web], is comprised of just two
functions and a convenient macro:

* `run` - runs your handler in a specific environment, responding to
  web requests matching a given host, port, path and virtual host. The
  handler may be either a Ring function, Servlet instance, or Undertow
  HttpHandler
* `run-dmc` - runs your handler in *Development Mode* (the 'C' is silent)
* `stop` - stops your handler[s]

The [immutant.web.middleware] namespace includes two Ring middleware
functions:

* `wrap-session` - enables session sharing among your Ring handler and
  its websockets, as well as automatic session replication when your
  app is deployed to a WildFly or EAP cluster.
* `wrap-development` - included automatically by `run-dmc`, this
  aggregates some middleware handy during development.

[WebSockets] are created using the [immutant.web.websocket] namespace,
which includes the following:

* `Channel` - a protocol for WebSocket interaction.
* `Handshake` - a protocol for obtaining attributes of the initial
  upgrade request
* `wrap-websocket` - middleware to attach websocket callback functions
  to a Ring handler

Finally, [immutant.web.undertow] exposes tuning options for Undertow,
should your app require that level of configuration.

## A sample REPL session

Now, let's fire up a REPL and work through some of the features of the
library.

If you haven't already, you should read through the [installation]
tutorial and require the `immutant.web` namespace at a REPL to follow
along:

<pre class="syntax clojure">(require '[immutant.web :refer :all])</pre>

### Common Usage

First, you'll need a [Ring] handler. If you generated your app using a
template from [Compojure], [Luminus], [Caribou] or some other
Ring-based library, yours will be associated with the `:handler` key
of your `:ring` map in your `project.clj` file. Of course, a far less
fancy handler will suffice:

<pre class="syntax clojure">(defn app [request]
  {:status 200
   :body "Hello world!"})
</pre>

To make the app available at <http://localhost:8080/>, do this:

<pre class="syntax clojure">(run app)</pre>

Which, if we make the default values explicit, is equivalent to this:

<pre class="syntax clojure">(run app {:host "localhost" :port 8080 :path "/"})</pre>

Or, since `run` takes options as either an explicit map or keyword
arguments (kwargs), this:

<pre class="syntax clojure">(run app :host "localhost" :port 8080 :path "/")</pre>

The options passed to `run` determine the URL used to invoke your
handler: `http://{host}:{port}{path}`

To replace your `app` handler with another, just call run again with
the same options, and it'll replace the old handler with the new:

<pre class="syntax clojure">(run (fn [_] {:status 200 :body "hi!"}))</pre>

To stop the handler, do this:

<pre class="syntax clojure">(stop)</pre>

Which is equivalent to this:

<pre class="syntax clojure">(stop {:host "localhost" :port 8080 :path "/"})</pre>

Or like `run`, if you prefer kwargs, this:

<pre class="syntax clojure">(stop :host "localhost" :port 8080 :path "/")</pre>

Alternatively, you can save the return value from `run` and pass it to
stop to stop your handler.

<pre class="syntax clojure">(def server (run app {:port 4242 :path "/hello"}))
...
(stop server)
</pre>

Stopping your handlers isn't strictly necessary if you're content to
just let the JVM exit, but it can be handy at a REPL.

### Virtual Hosts

The `:host` option denotes the IP interface to which the web server is
bound, which may not be publicly accessible. You can extend access to
other hosts using the `:virtual-host` option, which takes either a
single hostname or multiple:

<pre class="syntax clojure">(run app :virtual-host "yourapp.com")
(run app :virtual-host ["app.io" "app.us"])
</pre>

Multiple applications can run on the same `:host` and `:port` as long
as each has a unique combination of `:virtual-host` and `:path`.

### Advanced Usage

The `run` function returns a map that includes the options passed to
it, so you can thread `run` calls together, useful when your
application runs multiple handlers. For example,

<pre class="syntax clojure">(def everything (-> (run hello)
                  (assoc :path "/howdy")
                  (->> (run howdy))
                  (merge {:path "/" :port 8081})
                  (->> (run ola))))
</pre>

The above actually creates two Undertow web server instances: one
serving requests for the `hello` and `howdy` handlers on port 8080,
and one serving `ola` responses on port 8081.

You can stop all three apps (and shutdown the two web servers) like
so:

<pre class="syntax clojure">(stop everything)</pre>

Alternatively, you could stop only the `ola` app like so:

<pre class="syntax clojure">(stop {:path "/" :port 8081})</pre>

You could even omit `:path` since "/" is the default. And because ola
was the only app running on the web server listening on port 8081, it
will be shutdown automatically.

### Handler Types

Though the handlers you run will typically be Ring functions, you can
also pass any valid implementation of `javax.servlet.Servlet` or
`io.undertow.server.HttpHandler`. For an example of the former, here's
a very simple [Pedestal] service running on Immutant:

<pre class="syntax clojure">(ns testing.hello.service
  (:require [io.pedestal.service.http :as http]
            [io.pedestal.service.http.route.definition :refer [defroutes]]
            [ring.util.response :refer [response]]
            [immutant.web :refer [run]]))

(defn home-page [request] (response "Hello World!"))
(defroutes routes [[["/" {:get home-page}]]])
(def service {::http/routes routes})

(defn start [options]
  (run (::http/servlet (http/create-servlet service)) options))
</pre>

### Development Mode

The `run-dmc` macro resulted from a desire to provide a no-fuss way to
enjoy all the benefits of REPL-based development. Before calling
`run`, `run-dmc` will first ensure that your Ring handler is
var-quoted and wrapped in the `reload` and `stacktrace` middleware
from the `ring-devel` library (which must be included among your
`[:profiles :dev :dependencies]` in `project.clj`). It'll then open
your app in a browser.

Both `run` and `run-dmc` accept the same options. You can even mix
them within a single threaded call.

### WebSockets

Also included in the `org.immutant/web` library is the
[immutant.web.websocket] namespace, which includes the
`wrap-websocket` function that attaches a map of callback functions to
your Ring handler. Though it looks like Ring middleware, it actually
returns an `HttpHandler` instead of a function, so it must come last
in your middleware chain.

The valid websocket event keywords and their corresponding callback
signatures are as follows, where channel is an instance of the
`Channel` protocol, and handshake is an instance of `Handshake`:

<pre class="syntax clojure">  :on-message (fn [channel message])
  :on-open    (fn [channel handshake])
  :on-close   (fn [channel {:keys [code reason]}])
  :on-error   (fn [channel throwable])
</pre>

To create your websocket endpoint, pass the result from
`wrap-websocket` to `immutant.web/run`. Here's an example that
asynchronously returns the upper-cased equivalent of whatever message
it receives:

<pre class="syntax clojure">(ns whatever
  (:require [immutant.web             :as web]
            [immutant.web.websocket   :as ws]
            [ring.middleware.resource :refer [wrap-resource]]
            [ring.util.response       :refer [redirect]]
            [clojure.string           :refer [upper-case]]))

(defn create-websocket []
  (web/run
    (-> (fn [{c :context}] (redirect (str c "/index.html")))
      (wrap-resource "public")
      (ws/wrap-websocket {:on-message (fn [c m] (ws/send! c (upper-case m)))}))
    {:path "/ws"}))
</pre>

After running the above, a request to <http://localhost:8080/ws>
should return a 302 redirect to <http://localhost:8080/ws/index.html>.
Assuming the `wrap-resource` middleware can find `public/index.html`
in your classpath (typically in your project's `resources/` dir), a
`<script>` that attempts to open a WebSocket connection to
<ws://localhost:8080/ws> should work, and an upper-cased version of
any text the browser sends should be returned to it through that
WebSocket.

Note the `:path` argument applies to both the Ring handler and the
WebSocket, distinguished only by the request protocol, e.g. `http://`
vs `ws://`.

#### The WebSocket Handshake

Often, applications require access to data in the original upgrade
request associated with a WebSocket connection, perhaps for user
authentication or some such. That data is made available via the
`immutant.web.websocket/Handshake` protocol, an instance of which is
passed to the `:on-open` callback.

In particular, you can access all the headers sent in the upgrade
request, and if you're using the `wrap-session` middleware, you can
even access any session data stored on behalf of the user by the Ring
handler. Here's a contrived example in which the Ring handler stores a
random id in the session that is then sent back to the user when he
opens a WebSocket:

<pre class="syntax clojure">(ns whatever
  (:require [immutant.web             :as web]
            [immutant.web.websocket   :as ws]
            [immutant.web.middleware  :refer [wrap-session]]
            [ring.util.response       :refer [response]]))

(def callbacks {:on-open (fn [c h] (ws/send! c (:id (ws/session h))))}

(defn share-session-with-websocket []
  (web/run
    (-> (fn [{{:keys [id] :or {id (str (rand))}} :session}]
          (-> id response (assoc :session {:id id})))
      (wrap-session)
      (ws/wrap-websocket callbacks))
    {:path "/ws"}))
</pre>

## Feature Demo

We maintain a Leiningen project called the [Immutant Feature Demo]
demonstrating all the Immutant namespaces, including examples of
simple
[Web](https://github.com/immutant/feature-demo/blob/thedeuce/src/demo/web.clj)
and
[WebSocket](https://github.com/immutant/feature-demo/blob/thedeuce/src/demo/websocket.clj)
apps.

You should be able to clone it somewhere, cd there, and `lein run`.

Have fun!

[immutant.web]: https://projectodd.ci.cloudbees.com/job/immutant2-incremental/lastSuccessfulBuild/artifact/target/apidocs/immutant.web.html
[immutant.web.middleware]: https://projectodd.ci.cloudbees.com/job/immutant2-incremental/lastSuccessfulBuild/artifact/target/apidocs/immutant.web.middleware.html
[immutant.web.websocket]: https://projectodd.ci.cloudbees.com/job/immutant2-incremental/lastSuccessfulBuild/artifact/target/apidocs/immutant.web.websocket.html
[immutant.web.undertow]: https://projectodd.ci.cloudbees.com/job/immutant2-incremental/lastSuccessfulBuild/artifact/target/apidocs/immutant.web.undertow.html
[Undertow]: http://undertow.io/
[Ring]: https://github.com/ring-clojure/ring/wiki
[installation]: /tutorials/installation/
[Pedestal]: https://github.com/pedestal/pedestal
[Compojure]: https://github.com/weavejester/compojure
[Luminus]: http://www.luminusweb.net/
[Caribou]: http://let-caribou.in/
[WebSockets]: http://en.wikipedia.org/wiki/WebSocket
[Immutant Feature Demo]: https://github.com/immutant/feature-demo
