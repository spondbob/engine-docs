---
title: Data Privacy
description: Learn about what data Engine collects and how you can configure Engine to ignore PII and other sensitive information.
---

This page will walk you through what information Engine sees about your GraphQL service's request, what Engine's default behavior to handle request data is, and how you can configure Engine to the level of data privacy your team needs.

<h2 id="architecture">Engine Architecture</h2>

Engine is primarily a cloud service that ingests and stores performance metrics data from your server. There are two ways to get data into Engine:

1. Use **Apollo Server 2** (Node servers) and [configure performance metrics reporting](https://www.apollographql.com/docs/apollo-server/features/metrics.html#Apollo-Engine) by providing an Engine API key in your serer configuration.
1. Run the **Engine proxy** in front of your server and [install an Apollo tracing package](https://www.apollographql.com/docs/engine/setup-standalone.html#supported-servers) in your server.

#### Apollo Server 2

If you've set up Engine metrics forwarding using Apollo Server 2, Apollo Server will automatically start tracing the execution your requests and forwarding that information to the Engine service. Engine uses this trace data to reconstruct both operation-level timing data for given query shapes and field-level timing data for your overall schema. This data will become available for you to explore in the [Engine interface](https://engine.apollographql.com/).

Apollo Server will never forward the responses of your requests to Engine, but it will forward the shape of your request, the time it took each resolver to execute for that request, and the variables and headers of the request (configurable, [see below](./data-privacy.html#data-collection)).

#### Engine Proxy

This configuration option is primarily used to forward metrics to the Engine ingress from non-Node servers. The proxy is installed and run in your own environment on-prem as a [separately hosted process](./setup-standalone.html) that you route your client requests through. Apollo Server 1 users and other Node users users also have the option to run the Engine proxy as a [sidecar next to their Node server](./setup-node.html).

As your clients make requests to your server, the proxy reads response extension data to make caching decisions and aggregates tracing and error information into reports that it sends to the Engine ingress.

While the Engine proxy sees your client request data and service response data, it only collects and forwards data that goes into the reports you see in the Engine dashboards. All information that is sent from your on-premise proxy to the out-of-band Engine cloud service is configurable and can be turned off through configuration options. Data is aggregated and sent approximately every 5 seconds.


<h2 id="data-collection">Data collection</h2>

<h3 id="request">Request</h3>

In this section, we'll go over which parts of your GraphQL HTTP requests are collected by Engine.

#### Query operation string

Both Apollo Server 2 and the Engine proxy report the full operation string of your request to the Engine cloud service. Because of this, you should be careful to put any sensitive data like passwords and PII in the GraphQL variables object rather than in the operation string itself.

#### Variables

Both Apollo Server 2 and the Engine proxy will report your the query variables for each request to the Engine cloud service by default. This can be disabled in the following ways:

- **Apollo Server 2** – use the [`privateVariables` option](https://www.apollographql.com/docs/apollo-server/api/apollo-server.html#EngineReportingOptions) in your Apollo Server configuration for Engine.

- **Engine proxy** – use the [`privateVariables` option](./proxy-config.html#Reporting) in your proxy configuration, or prevent all variables from being reported with [`noTraceVariables` option](./proxy-config.html#Reporting).

<h4 id="http-headers">Authorization & Cookie HTTP Headers</h4>

Engine will **never** collect your application's `Authorization`, `Cookie`, or `Set-Cookie` headers and ignores these if received. Engine will collect all other headers from your request to show in the trace inspector, unless turned off with these configurations:

- **Apollo Server 2** – use the [`privateHeaders` option](https://www.apollographql.com/docs/apollo-server/api/apollo-server.html#EngineReportingOptions) in your Apollo Server configuration for Engine.
- **Engine Proxy** – use the [`privateHeaders` option](./proxy-config.html#Reporting) in your proxy configuration.

If you perform authorization in another header (like `X-My-API-Key`), be sure to add this to `privateHeaders` configuration. Note that unlike headers in general, this configuration option **is** case-sensitive.

<h3 id="response">Response</h3>

Let's walk through Engine's default behavior for reporting on fields in a typical GraphQL response:

```
// GraphQL Response
{
  "data": { ... },          // Never sent to the Engine cloud service
  "errors": [ ... ],        // Sent to Engine, used to report on errors for operations and fields.
  "extensions": {
    "tracing": { ... },     // Sent to Engine, used to report on performance data for operations and fields.
    "cacheControl": { ... } // Sent to Engine, used to determine cache policies and forward CDN cache headers.
  }
} 
```

#### `response.data`

Neither Apollo Server 2 nor the Engine proxy will ever send the contents of this to the Engine cloud service. The responses from your GraphQL service stay on-prem.

If you've configured whole query caching through the Engine proxy and Engine determines that a response it sees is cacheable, then the response will be stored in your [Engine cache](./caching.html#config.stores) (either in-memory in your proxy or as an external memcached you configure).

#### `response.errors`

If either Apollo Server 2 or the Engine proxy sees a response with an `"errors"` field, they will read the `message` and `locations` fields if they exist and report them to the Engine cloud service.

You can disable reporting errors to the out-of-band Engine cloud service like so:

- **Apollo Server 2** – you cannot disbale error reporting with Apollo Server 2 today, but we're tracking the issue [here](https://github.com/apollographql/apollo-server/issues/1613).
- **Engine proxy** – use the [`noTraceErrors` option](./proxy-config.html#Reporting) to disable sending error traces to the Engine cloud service.

#### Disable Reporting (Engine proxy)

We've added the option to disable reporting of proxy stats and response traces to the Engine cloud service so that integration tests can run without polluting production data.

To disable all reporting, use the [`disabled` option](./proxy-config.html#Reporting) for the Engine proxy.

<h2 id="policies" title="Policies and Agreements">Policies and Agreements</h2>

To learn about other ways that we protect your data, please read over our [Terms of Service](https://www.apollographql.com/policies/terms) and [Privacy Policy](https://www.apollographql.com/policies/privacy).
