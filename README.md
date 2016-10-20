# KafkaSSE

Kafka Consumer to HTTP SSE/EventSource

Uses [node-rdkafka](https://github.com/Blizzard/node-rdkafka) KafkaConsumer to
stream JSON messages to clients over HTTP in [SSE/EventSource format](https://www.w3.org/TR/eventsource/)

The `Last-Event-ID` and EventSource `id` field will be used to handle auto-resume
during client disconnects.  By using EventSource to connect to a KafkaSSE endpoint,
your client will automatically resume from where it left off if it gets disconnected
from the server.  Every message sent to the client will have an `id` field that will be
a JSON array of objects describing each latest topic, partition and offset seen by this client.
On a reconnect, this object will be sent back as the `Last-Event-ID` header, and will be used
by KafkaSSE to assign a KafkaConsumer to start from those offsets.

See also [Kasocki](https://github.com/wikimedia/kasocki).

## Usage

### HTTP server KafkaSSE set up
The kafka-sse module exports a function that wraps up handling an HTTP SSE request for Kafka topics.

```javascript
'use strict';
const kafkaSse = require('kafka-sse');
const server = require('http').createServer();

const options = {
    kafkaConfig: {'metadata.broker.list': 'mybroker:9092'}
}

server.on('request', (req, res) => {
    const topics = req.url.replace('/', '').split(',');
    console.log(`Handling SSE request for topics ${topics}`);
    kafkaSse(req, res, topics, options)
    // This won't happen unless client disconnects or kafkaSse encounters an error.
    .then(() => {
        console.log('Finished handling SSE request.');
    });
});

server.listen(6917);
console.log('Listening for SSE connections at http:/localhost:6917/:topics');
```

### NodeJS EventSource usage
```javascript
const EventSource = require('eventsource');
'use strict';
const topics = process.argv[2];
const port   = 6917

const url = `http://localhost:${port}/${topics}`;
console.log(`Connecting to Kafka SSE server at ${url}`);
let eventSource = new EventSource(url);

eventSource.onopen = function(event) {
    console.log('--- Opened SSE connection.');
};

eventSource.onerror = function(event) {
    console.log('--- Got SSE error', event);
};

eventSource.onmessage = function(event) {
    // event.data will be a JSON string containing the message event.
    console.log(JSON.parse(event.data));
};
```


## Errors

If an error is encountered during SSE client connection, a normal HTTP error response
will be returnred, along with JSON information about the error in the response body.
However, once the SSE connection has started, the HTTP response header will have already
been set to 200.  From that point on, errors are given to the client as `onerror` EventSource
events.  You must register an `onerror` function for your EventSource object to receive these.


## Notes on Kafka consumer state

In normal use cases, Kafka (and previously Zookeeper) handles consumer state.
Kafka keeps track of multiple consumer processes in named consumer groups, and
handles rebalancing of those processes as they come and go.  Kafka also
handles offset commits, keeping track of the high water mark each consumer
has reached in each topic and partition.

KafkaSSE is intended to be exposed to the public internet by enabling
web based consumers to use HTTP to consume from Kafka.  Since
the internet at large cannot be trusted, we would prefer to avoid allowing
the internet to make any state changes to our Kafka clusters.  KakfaSSE
pushes as much consumer state management to the connected clients as it can.

Offset commits are not supported.  Instead, latest subscription state is sent
as the EventSource `id` field with each event.  This information can be
used during connection initializion in the `Last-Event-ID` header
to specify the positions at which KafkaSSE should start consuming from Kafka.
`Last-Event-ID` should be an assignments array, of the form:

```javascript
[
    { topic: 'topicA', partition: 0, offset 12345 },
    { topic: 'topicB', partition: 0, offset 46666 },
    { topic: 'topicB', partition: 1, offset 45555 },
]
```

Consumer group management is also not supported.  Each new SSE client
corresponds to a new consumer group.  There is no way to parallelize
consumption from Kafka for a single connected client.  Ideally, we would not
register a consumer group at all with Kafka, but as of this writing
[librdkafka](https://github.com/Blizzard/node-rdkafka/issues/18) and
[blizzard/node-rdkafka](https://github.com/Blizzard/node-rdkafka/issues/18)
don't support this yet.  Consumer groups that are registered with Kafka
are named after the `x-request-id` header, or a uuid if this is not set, e.g.
`KafkaSSE-2a360ded-1da0-4258-bad5-90ce954b7c52`.

## node-rdkafka consume modes
The node-rdkafka client that KafkaSSE uses has
[several consume APIs](https://github.com/Blizzard/node-rdkafka#kafkakafkaconsumer).
KafkaSSE uses the [Standard Non flowing API](https://github.com/Blizzard/node-rdkafka#standard-api-1).


## Testing
Mocha tests require a running 0.9+ Kafka broker at `localhost:9092` with
`delete.topic.enable=true`.  `test/utils/kafka_fixture.sh` will prepare
topics in Kafka for tests.  `npm test` will download, install, and run
a Kafka broker.  If you already have one running locally, then
`npm run test-local` will be easier to run.

Note that there is a
[bug in librdkafka/node-rdkafka](https://github.com/edenhill/librdkafka/issues/775)
that keeps tests from shutting down once done.  This bug also has implications
for the number of consumers a process can run at once in its lifetime,
and will have to be resolved somehow before this is put into production.


## To Do

- tests for kafkaEventHandlers
- statsd + metrics
- Get upstream fix for https://github.com/Blizzard/node-rdkafka/issues/5
  this will need to be resolved before this can be used in any type of production
  setting.
- rdkafka statsd: https://phabricator.wikimedia.org/T145099
- filter, jsonschema? jsonschema query?

- Replace Kasocki references in package.json