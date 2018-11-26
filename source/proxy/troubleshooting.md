---
title: Setup troubleshooting
---

If you hit any issues in setting up Engine for your GraphQL service, we're here to help! First, follow these troubleshooting steps to check for any obvious issues. If these don't help, please submit a support ticket to <a href="mailto:support@apollographql.com">Apollo support</a> and we'll work with you to get you up and running!

<h2 id="sanity-checks" title="First steps">First Troubleshooting steps</h2>

<h3 id="check-server-support"> Check that you are on a supported GraphQL server </h3>

Check that your server is one of the supported GraphQL servers listed [here](apollo-tracing.html).

If it is, please make sure you're running the [currently tested version](https://github.com/apollographql/apollo-engine-js/blob/master/package.json) of Apollo Server and your Node HTTP server package (Express, Connect, Hapi, Koa, etc), and the latest released versions of the Engine and Apollo packages.

You can enter the following into the commandline to see the latest package version, or look in  `package.json`.

```
$ npm view apollo-engine version
```

<h3 id="proxy-debug-levels"> Set debug logging levels for the Proxy</h3>

Support may request that you set the Engine Proxy logging level to DEBUG or higher. These logs will be part of your GraphQL server logs (if Proxy is deployed with the `ApolloEngine` Node API) or in the Proxy process logs (if Proxy is deployed standalone).

```js
const engine = new ApolloEngine({
  logging: {
    level: 'DEBUG'   // Engine Proxy logging level. DEBUG, INFO, WARN or ERROR
  },
});
```

<h3 id="ensure-tracing-enabled"> Ensure you enabled Apollo Tracing </h3>

Test that you enabled Apollo Tracing by checking if your GraphQL server returns trace extensions in GraphQL responses when not executed through Engine. If it does, it's is a sign that Apollo Tracing is properly configured.

<h2 id="troubleshooting-faqs">Troubleshooting FAQs</h2>

**I'm getting an error saying “The query failed!”, how do I fix it?**

This may mean you need to upgrade an NPM package. Check that your package versions are all up-to-date. This also may mean a variety of other things. When this error is paired with a 503 error, the query did not receive an expected response.

**Why isn't data showing up in my dashboard?**

We recommend double-checking that the Engine API key for the correct service is specified in the `ApolloEngine` constructor.

**How do I check that the Engine Proxy is up and running?**

There is a health check URL at `[engine-frontend-url]/.well-known/apollo/engine-health`, which returns an HTTP status of 200 if the server is running.

**What is shown on the Engine Proxy logs?**

Each time the Engine proxy starts, you should see the following two lines in the logs indicating the Engine proxy is healthy:

```
INFO[0000] Started HTTP server.                          address="[::]:50485"
INFO[0000] Engine proxy started.                         version=2018.02-93-ge050c6b93
```

These lines say what port Engine is listening on and the internal version number for the Proxy. If you don't want to see them, set the log level to 'WARN'

```js
const engine = new ApolloEngine({
  logging: {
    level: 'WARN',
  },
});
```

<h2 id="get-support">Submit a support ticket</h2>

Please include the following when submitting an issue to our support team:

* Platform of GraphQL server
* Are you using `new ApolloEngine`, `new ApolloEngineLauncher`, or the Docker container?
* Engine configuration: arguments to `new ApolloEngine` or `new ApolloEngineLauncher`, or the JSON configuration file for the Docker container
* Platform of GraphQL server
* The query submitted and the full response

Submit your issue to <a href="mailto:support@apollographql.com">Apollo support</a> or you can join us in the public [#engine Slack Channel](https://www.apollographql.com/slack).
