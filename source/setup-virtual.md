---
title: Hosting Engine as a standalone server in PaaS Environments
sidebar_title: Engine standalone on PaaS
description: How to run the Engine container separately on a Platform as a Service like Heroku
---

If you want to run Engine separately from your GraphQL server, you can choose to host the Engine Docker container on a platform as a service such as Heroku, and use it as a proxy in front of your API.

To do this, you'll need to:

1. Instrument your server with a supported tracing agent that uses the Apollo Tracing format.
2. Configure and deploy the Engine proxy docker container.
3. Send requests to your service – you're all set up!

If you're using Node, we recommend running the Engine proxy with direction Web framework integration as opposed to within a Docker container inside of a separate app / dyno. In that case, follow the [Node setup instructions](setup-node.html). For all other platforms, the only available option is to run the proxy in a standalone docker container.

If you're interested in writing direct a direct Engine web framework integration for your platform, <a href="javascript:void(0);" onclick="Intercom('showNewMessage')">get in touch</a> with us!

For running on Heroku specifically, see our [Heroku Engine Proxy example repo](https://github.com/apollographql/engine-heroku-example).

<h2 id="enable-apollo-tracing" title="Enable Apollo Tracing">1. Enable Apollo Tracing</h2>

You will need to instrument your GraphQL server with a [tracing package](https://github.com/apollographql/apollo-tracing#supported-graphql-servers) that matches your server platform.  Engine relies on receiving data in this format to create its performance telemetry reports.

> Note: Apollo Server 2.x no longer requires a tracing package.  Instead, tracing is enabled automatically when an `ENGINE_API_KEY` is set in the environment, or when `engine: { apiKey: "XXX" }` is set to an Engine API key in the `ApolloServer` constructor options.

<h2 id="configure-proxy" title="Configure the Proxy">2. Configure the Proxy</h2>
<h3 id="get-api-key" title="Get your API Key">2.1 Get your API Key</h3>
First, get your Engine API key by creating a service on http://engine.apollographql.com/. You will need to log in and click "Add Service" to recieve your API key.

<h3 id="create-config-json" title="Create your Config.json">2.2 Follow the [Engine Heroku Example](https://github.com/apollographql/engine-heroku-example) to configure your Proxy</h3>
The standalone proxy uses a JSON file placed in the Docker container's root folder to get configuration information. Changes to the config.json will cause the proxy to adopt the new configuration without downtime.

> NOTE: For Virtual Hosted environments where the `PORT` is dynamically set in an environment variable named `$PORT`, you can leave out the `port` option. If your environment uses a different environment variable, you can name it with the `portFromEnv` option instead.

**Create a JSON configuration file:**

```
{
  "apiKey": "<ENGINE_API_KEY>",
  "logging": {
    "level": "INFO"
  },
  "origins": [
    {
      "http": {
        "url": "http://yourappname.herokuapp.com/graphql",
        "overrideRequestHeaders": {
          "Host": "yourappname.herokuapp.com"
        }
      }
    }
  ],
  "frontends": [
    {
      "host": "0.0.0.0",
      "port": "3000",
      "graphqlPaths": ["/graphql"]
    }
  ]
}
```

**Configuration options:**
1. `apiKey`: The API key for the Engine service you want to report data to.
2. `logging.level` : Logging level for the proxy. Supported values are `DEBUG`, `INFO`, `WARN`, `ERROR`.
3. `origin.http.url` : The URL for your GraphQL server.
4. `origins.http.overrideRequestHeaders.Host`: set to the origin hostname so virtual hosting systems can properly route to the origin.
5. `frontend.host` : The hostname the proxy should be available on.
6. `frontend.port` : The port the proxy should bind to.
7. `frontend.graphqlPaths` : The path for the proxy's GraphQL server. Defaults to `['/graphql']`.

For full configuration details see [Proxy config](proxy-config.html).

It does not matter where you choose to deploy and manage your Engine proxy. We run our own on Amazon's [EC2 Container Service](https://aws.amazon.com/ecs/).

We recognize that almost every team using Engine has a slightly different deployment environment, and encourage you to <a href="javascript:void(0);" onclick="Intercom('showNewMessage')">contact us</a> with feedback or for help if you encounter problems running the Engine proxy.

<h2 id="view-metrics-in-engine" title="View Metrics in Engine">3. View Metrics in Engine</h2>
Once your server is set up, navigate your new Engine service on https://engine.apollographql.com. Start sending requests to your GraphQL server to start seeing performance metrics!
