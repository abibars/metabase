# Response Handling

This describes the worst-case performance constraints that Metabase
deployments encounter, and how our response handling is tailored to
address those constraints.

## Performance Constraints

In the most extreme cases, there are dozens or hundreds of users
accessing dashboards that each load X cards. Each card requires the
execution of Y queries. These queries can take anywhere from
milliseconds to minutes to complete, creating significant back
pressure that must be managed:

* Metabase can't create too many threads because that will consume too
  much memory
* Metabase can't exceed database connection quotas
* Metabase must respond to health checks otherwise avoid getting
  killed by instance monitors
* Metabase should provide a good experience, which means that
  dashboards should load quickly

### Database constraints

Each database vendor and installation imposes different constraints on
the number of allowable concurrent queries. For example, BigQuery
[allows 100 concurrent interactive
queries](https://cloud.google.com/bigquery/quotas).

Metabase manages these constraints by providing per-driver connection
thread pools. By limiting the number of threads in the thread pool, it
limits the number of connections.

### Proxies

Metabase is sometimes installed behind proxies that will terminate a
request when Metabase doesn't send a response within some time frame,
usually 60 seconds.

TODO Is this the most accurate way to describe this?

### Health Monitoring / Instances getting killed

TODO AWS will kill Metabase instances when...

## Response Handling Strategies

### Async HTTP Responses

Many endpoints have async handlers. We use async handlers to reduce
thread consumption.

Async and synchronous handlers differ in two ways:

* Whether a single thread is dedicated to the request for its entire
  duration
* The signature of the (Clojure ring) request handler

Ring handlers are typically synchronous because synchronous handlers
are sufficient for the majority of use cases. Synchronous handlers are
a function that takes a request map as the only arguments, and returns
a response map:

```clojure
(fn [request] {:status 200 :body "json blob"})
```

The HTTP server that Ring is adapting will dedicate a thread to
handling this request until a response is returned.

This becomes problematic in Metabase because a request can involve
querying a resource that has high latency for any number of reasons:
the network might be slow, the query might have complex joins,
etc. Consuming threads when waiting leads to poor performance because
it increases memory consumption and can eventually lead to thread
starvation (when there are so many threads that no thread is able to
get scheduled enough time to actually complete its work).

Async handlers help by off-loading the handler from a thread while the
handler generates a response, freeing the thread for other work or
garbage collection. Async handlers are normally written as [functions
that take three
arguments](https://www.booleanknot.com/blog/2016/07/15/asynchronous-ring.html):

```clojure
(fn [request response raise] (response {:status 200 :body "json blob"}))
```

The handler's return value is not used for the response. Instead, you
use the `response` function to send a response.

### Delayed Responses

Since some endpoints (both async and synchronous) might take longer
than a minute to generate a response, and proxies or Heroku might kill
a HTTP connection if there's no activity for a minute, some endpoints
implement a strategy to stream newlines until a JSON response is
ready. (JSON ignores leading newlines.)

This doesn't necessarily improve performance because it doesn't change
the resource consumption needed to eventually produce a JSON
response. Upcoming changes will improve performance by using
transducers for streaming responses to reduce resource consumption.

How does Metabase accomplish this? Ring responses are `OutputStreams`,
and you tell ring how to create an output stream by implementing the
`StreamableResponseBody` protocol. Out of the box, this is implemented
for seqs, input streams, files, and strings. Metabase implements
`StreamableResponseBody` for core.async channels, and that
implementation includes the newline streaming.

Async request handling and delayed response handling are independent
of each other. It's possible to handle a request asynchronously by
returning a response body that isn't implementing a delayed response,
and it's possible to implement a delayed response without handling the
request asynchronously.

### Query Processor

TODO this probably deserves its own document

### Thread Pools

* Per-driver thread pools
* core.async has its own thread pool