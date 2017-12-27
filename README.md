# Envoy Node


[![Travis](https://api.travis-ci.org/Tubitv/envoy-node.svg?branch=master)](https://travis-ci.org/Tubitv/envoy-node) [![Coverage Status](https://coveralls.io/repos/github/Tubitv/envoy-node/badge.svg?branch=master)](https://coveralls.io/github/Tubitv/envoy-node?branch=master) [![npm version](https://img.shields.io/npm/v/envoy-node.svg)](https://www.npmjs.com/package/envoy-node)  ![npm license](https://img.shields.io/npm/l/envoy-node.svg)

This is a boilerplate to help you adopt [Envoy](https://github.com/envoyproxy/envoy).

There are multiple ways to config Envoy, one of the convenience way to mange different egress traffic is route the traffic by hostname (using [virtual hosts](https://www.envoyproxy.io/docs/envoy/v1.5.0/api-v1/route_config/vhost.html)). By doing so, you can use one egress port for all your egress dependencies:

```yaml
listener:
- address: tcp://127.0.0.1:12345
  filters:
  - name: http_connection_manager
    config:
      codec_type: auto
      use_remote_address: true
      stat_prefix: my-service.egress
      route_config:
        virtual_hosts:
        - name: foo_cluster
          domains:
          - foo.service:10080 # Do not miss the port number here
          routes:
          - prefix: /
            cluster: foo_cluster
        - name: bar_cluster
          domains:
          - bar.service:10081 # Do not miss the port number here
          routes:
          - prefix: /
            cluster: bar_cluster
      filters:
      - name: router
        config:
          dynamic_stats: true
      tracing:
        operation_name: egress
      access_log:
      - path: /tmp/envoy.my-service.egress.log
        filter:
          type: not_healthcheck
```

But it will bring you new problem, your code is becoming verbose:

1. routing traffic to `127.0.0.1:12345` where egress port is listening
2. setting host headers for each request
3. propagating the tracing information

And this library is going to help you deal with these things elegantly.

First, let's tell the library where the egress port is binding. A recommended way is to set the information on the ingress header by [request_headers_to_add](https://www.envoyproxy.io/docs/envoy/v1.5.0/api-v1/route_config/route_config#config-http-conn-man-route-table-add-req-headers):

```yaml
request_headers_to_add:
- key: x-tubi-envoy-egress-port
  value: "12345"
- key: x-tubi-envoy-egress-addr
  value: 127.0.0.1
```

You can also set this by the constructor parameters of `EnvoyContext`.

## High level APIs

### HTTP

For HTTP, you can new the client like this:

```js
const { EnvoyHttpClient, HttpRetryOn } = require("envoy-node");

async function awesomeAPI(req, res) {
  const client = new EnvoyHttpClient(req.headers);
  const url = `http://foo.service:10080/path/to/rpc`
  const request = {
    message: "ping",
  };
  const optionalParams = {
    // timeout 1 second
    timeout: 1000,
    // envoy will retry if server return HTTP 409 (for now)
    retryOn: [HttpRetryOn.RETRIABLE_4XX],
    // retry 3 times at most
    maxRetries: 3,
    // each retry will timeout in 300 ms
    perTryTimeout: 300,
  };
  const serializedJsonResponse = await client.post(url, request, optionalParams);
  res.send({ serializedJsonResponse });
  res.end();
}
```

### gRPC

For gRPC, you can new the client like this:

#### General RPC

```js
const grpc = require("grpc");
const { envoyProtoDecorator, GrpcRetryOn } = require("envoy-node");

const PROTO_PATH = __dirname + "/ping.proto";
const Ping = grpc.load(PROTO_PATH).test.Ping;

// the original client will be decorated as a new class
const PingClient = envoyProtoDecorator(Ping);

async function awesomeAPI(call, callback) {
  const client = new PingClient("bar.service:10081", call.metadata);
  const request = {
    message: "ping",
  };
  const optionalParams = {
    // timeout 1 second
    timeout: 1000,
    // envoy will retry if server return DEADLINE_EXCEEDED
    retryOn: [GrpcRetryOn.DEADLINE_EXCEEDED],
    // retry 3 times at most
    maxRetries: 3,
    // each retry will timeout in 300 ms
    perTryTimeout: 300,
  };
  const response = await client.pathToRpc(request, optionalParams);
  callback(undefined, { remoteResponse: response });
}
```

#### Streaming API

But they are also decorated to send the Envoy context. You can also specify the optional params (the last one) for features like `timeout` / `retryOn` / `maxRetries` / `perTryTimeout` provided by Envoy.

**NOTE**:

1. For streaming API, they are not implemented as `async` signature.
2. The optional params (`timeout` etc.) is not tested and Envoy is not documented how it deal with streaming.

##### Client streaming

```js
const stream = innerClient.clientStream((err, response) => {
  if (err) {
    // error handling
    return;
  }
  console.log("server responses:", response);
});
stream.write({ message: "ping" });
stream.write({ message: "ping again" });
stream.end();
```

##### Sever streaming

```js
const stream = innerClient.serverStream({ message: "ping" });
stream.on("error", error => {
  // handle error here
});
stream.on("data", (data: any) => {
  console.log("server sent:", data);
});
stream.on("end", () => {
  // ended
});
```

##### Bidirectional streaming

```js
const stream = innerClient.bidiStream();
stream.write({ message: "ping" });
stream.write({ message: "ping again" });
stream.on("error", error => {
  // handle error here
});
stream.on("data", (data: any) => {
  console.log("sever sent:", data);
});
stream.on("end", () => {
  stream.end();
});
stream.end();
```

## Low level APIs

If you want to have more control of your code, you can also use the low level APIs of this library:

```js
const { envoyFetch, EnvoyContext, EnvoyHttpRequestParams, EnvoyGrpcRequestParams } = require("envoy-node");

// ...

const context = new EnvoyContext(
  headerOrMetadata,
  // specify port if we cannot indicate from
  // - `x-tubi-envoy-egress-port` header or
  // - environment variable ENVOY_DEFAULT_EGRESS_PORT
  envoyEgressPort,
  // specify address if we cannot indicate from
  // - `x-tubi-envoy-egress-addr` header or
  // - environment variable ENVOY_DEFAULT_EGRESS_ADDR
  envoyEgressAddr
);

// for HTTP
const params = new EnvoyHttpRequestParams(context, optionalParams);
envoyFetch(params, url, init /* init like original node-fetch */)
  .then(res => {
    console.log("envoy tells:", res.overloaded, res.upstreamServiceTime);
    return res.json(); // or res.text(), just use it as what node-fetch returned
  })
  .then(/* ... */)

// for gRPC
const client = new Ping((
  `${context.envoyEgressAddr}:${context.envoyEgressPort}`, // envoy egress port
  grpc.credentials.createInsecure()
);
const requestMetadata = params.assembleRequestMeta()
client.pathToRpc(
  request,
  requestMetadata,
  {
    host: "bar.service:10081"
  },
  (error, response) => {
    // ...
  })

```

Check out the [detail document](https://tubitv.github.io/envoy-node/) if needed.

## For dev and test

If you are developing the application, you may probably do not have Envoy running. You may want to call the service directly:

Either:

```js
new EnvoyContext(headers, undefined, undefined, true)
```

or:

```shell
export ENVOY_DIRECT_MODE=true # 1 works as well
```

## Contributing

For developing or running test of this library, you probably need to:

1. have an envoy binary in your `PATH`, or:
  ```shell
  $ npm run download-envoy
  $ export PATH=./node_modules/.bin/:$PATH
  ```
2. there is a bug in gRPC's typing, run the following to fix it:
  ```shell
  $ npm run fix-grpc-typing-bug
  ```
3. to commit your code change:
  ```shell
  $ git add . # or the things you want to commit
  $ npm run commit # and answer the commit message accordingly
  ```

## License

MIT

## Credits

- this library is init by alexjoverm's [typescript-library-starter](https://github.com/alexjoverm/typescript-library-starter)

- Thanks [@mattklein123](https://github.com/mattklein123) and Envoy community for questions and answers.
