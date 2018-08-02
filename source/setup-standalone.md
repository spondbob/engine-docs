---
title: Set up Engine with any GraphQL server
sidebar_title: Non-Node servers
description: Get Engine running with your Ruby, Java, Scala, or Elixir server.
---

The easiest way to get Engine running is our [direct integration with Node.js web frameworks and Apollo Server](./setup-node.html).  But it takes only a little more setup to get Engine working with any GraphQL API by running it as a proxy that processes requests before they hit your server.

To get started with Engine, you will need to:
1. Instrument your GraphQL server to add Apollo Tracing extension data to its responses.
2. Get your Engine API key.
3. Configure and deploy the Engine Proxy using Docker or npm.

<h2 id="supported-servers">Instrument your server</h2>

A few Engine features, like [automatic persisted queries](./auto-persisted-queries.html) and [query batching](./query-batching.html) work with any GraphQL server, but you'll get the most value out of Engine if your server to supports the [Apollo Tracing](./apollo-tracing.html) GraphQL extension, which is required for [performance tracing](./performance.html) and [error tracking](./error-tracking.html).  Additionally, the  [Apollo Cache Control](https://github.com/apollographql/apollo-cache-control) extension is required for [response caching](./caching.html). These extensions are open specifications that can be implemented by any GraphQL server.

The following GraphQL servers support at least one of the Apollo GraphQL extensions.

1. **Node** with [Apollo Server](https://www.apollographql.com/docs/apollo-server/) natively supports Apollo Tracing and Apollo Cache Control. See [Node setup instructions](./setup-node.html) for a more streamlined Node setup option.
2. **Ruby** with [GraphQL-Ruby](http://graphql-ruby.org/) supports Apollo Tracing with the [apollo-tracing-ruby](https://github.com/uniiverse/apollo-tracing-ruby) gem.
3. **Java** with [GraphQL-Java](https://github.com/graphql-java/graphql-java) natively supports Apollo Tracing. [Read the docs about using Apollo Tracing.](http://graphql-java.readthedocs.io/en/latest/instrumentation.html#apollo-tracing-instrumentation)
4. **Scala** with [Sangria](https://github.com/sangria-graphql/sangria) supports Apollo Tracing with [sangria-slowlog](https://github.com/sangria-graphql/sangria-slowlog#apollo-tracing-extension) project.
5. **Elixir** with [Absinthe](https://github.com/absinthe-graphql/absinthe) supports Apollo Tracing with the [apollo-tracing-elixir](https://github.com/sikanhe/apollo-tracing-elixir) package.

See the individual package documentation to learn how to enable Apollo Tracing for your platform.

You can test that you've correctly enabled Apollo Tracing by running any query against your API using GraphiQL.

The `"tracing"` field should now be returned as part of the response extensions like below. Don't worry, this data won't make it to your frontend once you've set up Engine, because Engine will filter it out.

```js
{
  "data": { ... },
  "extensions": {
    "tracing": { ... }
  }
}
```

If you see the `tracing` data in the result, move on to the next step!

<h2 id="get-api-key">Get your API Key</h2>

[Log into Apollo Engine](http://engine.apollographql.com/) via your browser and create a service to get an API key. We'll put it in our Engine configuration in the next step.

<h2 id="configure-proxy">Run the proxy</h2>

Engine uses a proxy component written in Go that runs inside your infrastructure. This component is designed to allow all of your requests and responses to pass through, while doing things like extracting trace data, caching results, and more. It's designed to handle large volumes of traffic comfortably, without overloading. The Engine proxy component doesn't rely on accessing the Engine cloud service to function.

Apollo distributes the Engine proxy in two forms: as an npm package and as a Docker container. You can use either distribution to run a standalone proxy in front of your GraphQL server, depending on whether it's easier for you to run Node programs or Docker containers.

<h3 id="apollo-engine-launcher">Running a standalone Proxy using Node</h3>

Even though your GraphQL server may not be implemented in Node, you may find it easier to run a tiny Node program in your hosting environment than to run a Docker container. This deployment method is for you. In addition to the [`ApolloEngine` API](./setup-node.html#api-apollo-engine) that allows you to directly integrate Engine with a Node GraphQL server, the `apollo-engine` npm package also contains the `ApolloEngineLauncher` API, which simply runs the Engine proxy with a given configuration.

First, install the `apollo-engine` package from npm:

```bash
npm install --save apollo-engine
```

Then write a small Node program that uses it.

```js
const { ApolloEngineLauncher } = require('apollo-engine');

// Define the Engine configuration.
const launcher = new ApolloEngineLauncher({
  // Enter your API key from step 2 above. You can also provide this
  // in the ENGINE_API_KEY environment variable.
  apiKey: 'API_KEY_HERE',
  origins: [{
    http: {
      // The URL that the Proxy should use to connect to your
      // GraphQL server.
      url: 'http://localhost:4000/api/graphql',
    },
  }],
  // Tell the Proxy on what port to listen, and which paths should
  // be treated as GraphQL instead of transparently proxied as raw HTTP.
  // You can leave out the frontend section if you want: the default for
  // 'port' is process.env.PORT, and the default for endpoints is
  // ['/graphql'].
  frontends: [{
    port: 3000,
    endpoints: ['/api/graphql'],
  }],
});

// Start the Proxy; crash on errors.
launcher.start().catch(err => { throw err; });
```
> Note: Every deployment has its unique needs, and we provide a variety of configuration options to fulfill them. For example, if your origin GraphQL server is running in a virtual-hosted environment (e.g. Heroku, AWS), you may need to override the `Host` header sent to HTTP origins. For more details and instruction on configuration options, please see the [proxy config docs](./proxy-config.html).


Now run this program with Node. The Proxy should start up and accept connections at http://localhost:3000 and forward all requests to your server at http://localhost:4000. Load GraphiQL through Engine at http://localhost:3000/graphiql (or wherever you have configured your app to serve GraphiQL) and run any query. You should no longer see the `tracing` data in the result since Engine is now consuming it! Check the Engine UI for your new service, and you should see it confirm that the setup worked.

If you see the above, congratulations, you're up and running!

In general, the argument to `new ApolloEngineLauncher()` is the same as the argument Node GraphQL users pass to `new ApolloEngine()`, except that you need to specify the origin's HTTP URL yourself with `new ApolloEngineLauncher()`, and the frontend `port` and `endpoints` (if you're not using the default values of `process.env.PORT` and `['/graphql']`) are specified inside the constructor instead of as options to `listen()`.  (We use the name `endpoints` in `ApolloEngineLauncher` instead of `graphqlPaths`; this is an older name and we intend to update it to consistently say `graphqlPaths`.)

<h3 id="docker">Running a standalone Proxy with Docker</h3>

The Engine Proxy is also distributed as a Docker image that you can deploy and manage separate from your server. It does not matter where you choose to deploy and manage your Engine Proxy, though it's more efficient if it is located on the same machine or network as your GraphQL server. We run our own on Amazon's [EC2 Container Service](https://aws.amazon.com/ecs/) and on Google's [Kubernetes Engine](https://cloud.google.com/kubernetes-engine/).

The Docker container distribution of Engine Proxy is configured using a JSON configuration file. The contents of the config file are the same as the argument to [`new ApolloEngineLauncher()`](#api-apollo-engine-launcher) and [`new ApolloEngine()`](./setup-node.html#api-apollo-engine). As they are JSON files, all object keys must be quoted, and trailing commas and comments are not allowed, but otherwise any reference in our docs to options passed to [`new ApolloEngine()`] translates directly into the engine config file. Like with `ApolloEngineLauncher`, you need to specify your GraphQL server's origin http URL (or other origin type like [Lambda](./setup-lambda.html)) inside the config file, and you need to specify the frontend port and GraphQL paths inside the config file rather than separately (if you're not using the default values of `process.env.PORT` and `['/graphql']`).

To get started with the Docker container distribution of Engine Proxy, write this in `engine-config.json`:

```js
{
  "apiKey": "API_KEY_HERE",
  "origins": [{
    "http": {
      "url": "http://localhost:4000/api/graphql"
    }
  }],
  "frontends": [{
    "port": 3000,
    "endpoints": ["/api/graphql"]
  }]
}
```
> Note: Every deployment has its unique needs, and we provide a variety of configuration options to fulfill them. For example, if your origin GraphQL server is running in a virtual-hosted environment (e.g. Heroku, AWS), you may need to override the `Host` header sent to HTTP origins. For more details and instruction on configuration options, please see the [proxy config docs](./proxy-config.html).

Make sure you have a working [Docker installation](https://docs.docker.com/engine/installation/), and type the following lines in your shell:

{% codeblock %}
$ ENGINE_PORT=3000
$ docker run --env "ENGINE_CONFIG=$(cat engine-config.json)" -p "${ENGINE_PORT}:${ENGINE_PORT}" gcr.io/mdg-public/engine:{% proxyDockerVersion %}
{% endcodeblock %}

This will run the Engine Proxy via Docker, routing port 3000 inside the container to port 3000 outside the container. (You can also pass `--net=host` instead of the `-p 3000:3000` to just allow the Proxy direct access to your host's network.)

We use [semver](https://semver.org/) to name Engine Proxy release versions, and we release version 1.2.3 under the tags `1.2.3`, `1.2`, and `1`.  If you want to pin to a precise version, use the `1.2.3` tag. If you'd like to take patch upgrades but not minor upgrades, use the `1.2` tag. If you'd like to take minor upgrades, use the `1` tag.

The Proxy should start up and accept connections at http://localhost:3000 and forward all requests to your server at http://localhost:4000. Load GraphiQL through Engine at http://localhost:3000/graphiql (or wherever you have configured your app to serve GraphiQL) and run any query. You should no longer see the `tracing` data in the result since Engine is now consuming it! Check the Engine UI for your new service, and you should see it confirm that the setup worked.

In general, the Engine config file is the same as the argument Node GraphQL users pass to `new ApolloEngine()`, except that you need to specify the origin's HTTP URL yourself with the engine config file, and the frontend `port` and `endpoints` (if you're not using the default values of `process.env.PORT` and `['/graphql']`) are specified inside the config file instead of as options to `listen()`. (Plus, it needs to be valid JSON: quoted keys, no trailing commas, no comments, etc.) The semantics of the Engine config file are identical to the argument to `new ApolloEngineLauncher()`.  (We use the name `endpoints` in the config JSON file instead of `graphqlPaths`; this is an older name and we intend to update it to consistently say `graphqlPaths`.)

You can find the complete set of configuration that Engine accepts in the [full API docs](./proxy-config.html) page, and some commonly-used fields worth knowing about are described below in the [`new ApolloEngineLauncher()` docs](#api-apollo-engine-launcher).

We recognize that almost every team using Engine has a slightly different deployment environment, and encourage you to <a href="javascript:void(0);" onclick="Intercom('showNewMessage')">contact us</a> with feedback or for help if you encounter problems running the Engine Proxy.

<h2 id="api">API</h2>

If you need additional configuration, you've come to the right place.

<h3 id="api-apollo-engine-launcher" title="ApolloEngineLauncher">new ApolloEngineLauncher(config)</h3>

The `ApolloEngineLauncher` class is a simple Node API for running the Engine Proxy on your machine. It doesn't integrate with Node web frameworks: for that you want [`ApolloEngine`](./setup-node.html#api-apollo-engine).

This is where you pass in configuration for how Engine should work. Even though you set up Engine as an npm package, it's actually a Go binary, which enables it to do things that would be harder to do in a performant way with purely Node.js code, while working the same way across many platforms.

To get started, the only configuration fields you need to specify are `apiKey` (the Engine API key copied from the Engine website) and `origins.http.url` (your GraphQL server's locally accessible URL).  (You can also provide the API key in the `ENGINE_API_KEY` environment variable.)

This configuration is identical to the argument to `new ApolloEngine()`, which is the API most of our docs describe, except that you must specify your GraphQL server's origin http URL (or other origin type like [Lambda](./setup-lambda.html)) inside the config argument, and you need to specify the frontend port and GraphQL paths inside the config argument rather than separately (if you're not using the default values of `process.env.PORT` and `['/graphql']`).

You can find the complete set of configuration that Engine accepts in the [full API docs](./proxy-config.html) page, but here are some commonly-used fields worth knowing about:

```js
const launcher = new ApolloEngineLauncher({
  // One of only two mandatory fields.
  apiKey: 'my-api-key',

  // Specify behavior for how the Engine Proxy should connect to the
  // GraphQL origin (your Node GraphQL server). While the Proxy does
  // support multiple origins, most users will only put one origin
  // in this array.
  origins: [{
    // HTTP-specific options.
    http: {
      // The URL that the Proxy should use to connect to your GraphQL
      // server. One of only two mandatory fields (unless you are
      // using a Lambda origin instead of http).
      url: 'http://localhost:4000/api/graphql',
      // The Engine Proxy will set the following request headers on
      // all requests it makes to your GraphQL server.
      overrideRequestHeaders: {
        // By default, the Engine Proxy preserves the Host header
        // from the user's request.
        host: 'mysite.example.com',
      },

      // If your HTTP server has more than one GraphQL server hosted
      // on different paths, and you've listed them all in your frontend's
      // endpoints field, set this to true and change the URL above
      // to your site's root (eg, 'http://localhost:4000') to tell Engine
      // Proxy to forward each of them to the matching path on this site.
      // (Otherwise, the Proxy will send all requests to the URL specified
      // above no matter which original URL they came from.)
      useFrontendPath: false,
    }

    // The options below apply both to HTTP origins and other origin
    // types such as Lambda.

    // If you are using the apollo-link-batch-http package to combine
    // multiple GraphQL requests into a single HTTP request, set this
    // to ensure that the Engine Proxy sends them to your Node web
    // server as a single request as well. (If you don't set this,
    // the Proxy will split it apart into individual HTTP requests.)
    supportsBatch: true,
    // Amount of time to wait for your Node GraphQL server to return
    // a response before timing out. Defaults to "30s". Specified as
    // a string containing a number and a unit such as "ms" or "s".
    requestTimeout: "60s",
  }],

  // Specify behavior for how the Engine Proxy listens as an HTTP
  // server. While the Proxy does support multiple listening
  // frontends, most users will only put one frontend in this array.
  frontends: [{
    // The port on which the Engine Proxy will listen.
    port: 3000,
    // The URL paths which the Engine Proxy will treat as GraphQL
    // instead of proxying transparently without parsing. Defaults to
    // ['/graphql'].
    endpoints: ['/api1/graphql', '/api2/graphql'],
    // The interface on which the Engine Proxy will accept connections.
    // Defaults to listening on all interfaces. For example, if your
    // machine is publically accessible on the Internet but you only
    // want processes on your machine (like nginx) to be able
    // to connect to the Engine Proxy, specify the "loopback address"
    // '127.0.0.1'.
    host: '127.0.0.1',
    // Specify behavior for handling GraphQL extensions in GraphQL
    // responses.
    extensions: {
      // Extensions which will never be returned to clients.
      // Defaults to ['tracing'].
      blacklist: ['tracing', 'cacheControl'],
      // Extensions to only return to clients if the client requests
      // them.  Defaults to `['tracing', 'cacheControl']`.
      strip: ['tracing'],
    },
  }],

  // Resize the default in-memory cache.
  stores: [{
    inMemory: {
      cacheSize: 104857600,  // 100 MB; defaults to 50MB.
    },
  }],
  sessionAuth: {
    // Use the value of the 'session-id' cookie to isolate responses
    // in the private full query cache from those from other sessions.
    cookie: 'session-id',
  },
  queryCache: {
    // Turn off the public full query cache. The cache is only used for
    // responses with the 'cache-control' GraphQL extension.
    publicFullQueryStore: 'disabled',
  },

  logging: {
    // Only show warnings and errors, not info messages.
    level: 'WARN',
  },
});
```

You can also specify config as a filename pointing to a JSON file.


<h3 id="api-launcher.start" title="launcher.start()">await launcher.start(options)</h3>

The `start` method is what actually starts the Engine Proxy. It's an async method, so either call it from within another async function with `await`, or handle the Promise that it returns.

When you run this method, it starts up Engine Proxy and waits for it to successfully listen for connections. The Promise it returns is resolved when it is listening. If it crashes due to invalid config, the Promise will reject. If it does not manage to start listening in 5 seconds (a very generous timeout), the Promise will reject.

If the Proxy exits for any reason (other than invalid config) during its execution, the launcher will restart it. When this happens, the launcher will emit a `restarting` event with more information, or log a message if there are no `restarting` listeners.

`start` has some optional options. Note that these are the same as the options in the `launcherOptions` option to [`ApolloEngine.listen()`](./setup-node.html#api-engine.listen)... now you know why that option has that name!

- `startupTimeout` (number): How many milliseconds to wait for the Engine Proxy to successfully start up and listen before timing out. Defaults to 5000 (5 seconds). Set to 0 to wait forever.
- `proxyStderrStream` ([writable stream](https://nodejs.org/api/stream.html#stream_writable_streams)): By default, the Engine Proxy binary's standard error stream is written to your Node process's standard error. Set this to a writable stream to process it in a different way.  (By default, the Engine Proxy writes its logs to standard error at log level `INFO`. If all you want to do is hide the `INFO` logs, try configuring the log level instead.)
- `proxyStdoutStream` ([writable stream](https://nodejs.org/api/stream.html#stream_writable_streams)): By default, the Engine Proxy binary's standard output stream is written to your Node process's standard output. Set this to a writable stream to process it in a different way.  (By default, the Engine Proxy does not write anything to standard output.)
- `processCleanupEvents` (array of strings): By default, `launcher.start()` registers handlers for the `exit`, `uncaughtException`, `SIGINT`, `SIGTERM`, and `SIGUSR2` [events on `process`](https://nodejs.org/api/process.html#process_process_events) to kill the Engine Proxy process (and, for the signal handlers, to wait for it to exit before resignaling the Node process). You can customize the list of events by specifying the event names as this option, if you want Engine to handle more signals or to let you do something different on the default events.


<h3 id="api-launcher.stop" title="launcher.stop()">await launcher.stop()</h3>

The `stop` method stops a running instance of the Engine Proxy and does not restart it. It's an async method, so either call it from within another async function with `await`, or handle the Promise that it returns. The Promise is resolved once the Proxy has exited.
