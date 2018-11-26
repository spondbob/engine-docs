---
title: Serverless
---

To get started with Engine for serverless environments, you will need to:
1. Instrument your function to respond with a tracing extension that follows the Apollo Tracing format.
2. Configure and deploy the Engine proxy docker container _separately_ as a standalone server. Because Engine proxy is stateful, it should not be deployed along with your cloud function, but separately.
3. Send requests to your service –-- you're all set up!

>Note: The remainder of these docs are framed specifically for AWS Lambda, for which we have a special "origin" type, but other cloud functions are supported with the standard HTTP invocation. For non-AWS cloud functions, see [the standalone docs](https://www.apollographql.com/docs/engine/setup-standalone.html#apollo-engine-launcher) for instructions on setting up the Engine proxy as a standalone API Gateway to your cloud function.

The Engine proxy will invoke the Lambda function as if it was called from [API Gateway](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-set-up-simple-proxy.html#api-gateway-simple-proxy-for-lambda-input-format), and the function should return a value suitable for [API Gateway](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-set-up-simple-proxy.html#api-gateway-simple-proxy-for-lambda-output-format).

We suggest using NodeJS, but any runtime supported by Lambda can be used.

**Supported Node servers:** [Apollo Server](https://github.com/apollographql/apollo-server) (Express, Hapi, Koa, Restify, and Lambda)

The only available option for running the Engine proxy with a function on Lambda is to run the proxy in a standalone docker container. The Proxy is required as it is responsible for capturing, aggregating and then sending to Engine the trace data from each Lambda instance GraphQL response.

<h2 id="enable-apollo-tracing" title="Enable Apollo Tracing">Enable Apollo Tracing</h2>

You will need to instrument your Lambda function with a tracing package that follows the [Apollo Tracing](https://github.com/apollographql/apollo-tracing) format. Engine relies on receiving data in this format to create its performance telemetry reports.

For Node with Apollo Server, just pass the `tracing: true` option to the Apollo Server graphql integration function, just like in our [Node setup instructions](./setup-node.html#apollo-server-config).

<h2 id="configure-proxy" title="Configure the Proxy">Configure the Proxy</h2>

<h3 id="get-api-key" title="Get your API Key">Get your API Key</h3>

First, get your Engine API key by creating a service on https://engine.apollographql.com/. You will need to log in and click "Add Service" to recieve your API key.

<h3 id="create-config-json" title="Create your Config.json">Create your Proxy's Config.json</h3>

The proxy uses a JSON object to get configuration information.

**Create a JSON configuration file:**

```
{
  "apiKey": "<ENGINE_API_KEY>",
  "logging": {
    "level": "INFO"
  },
  "origins": [
    {
      "lambda": {
          "functionArn":"arn:aws:lambda:xxxxxxxxxxx:xxxxxxxxxxxx:function:xxxxxxxxxxxxxxxxxxx",
          "awsAccessKeyId":"xxxxxxxxxxxxxxxxxxxx",
          "awsSecretAccessKey":"xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
      }
    }
  ],
  "frontends": [
    {
      "host": "0.0.0.0",
      "port": 3001,
      "endpoints": ["/graphql"]
    }
  ]
}
```

**Configuration options:**

1. `apiKey` : The API key for the Engine service you want to report data to.
2. `logging.level` : Logging level for the proxy. Supported values are `DEBUG`, `INFO`, `WARN`, `ERROR`.
3. `origin.lambda.functionArn` : The Lambda function to invoke, in the form: `arn:aws:lambda:xxxxxxxxxxx:xxxxxxxxxxxx:function:xxxxxxxxxxxxxxxxxxx`
4. `origin.lambda.awsAccessKeyId` : Your Access Key ID. If not provided the proxy will attempt `AWS_ACCESS_KEY_ID`/`AWS_SECRET_KEY` environment variables, and EC2 instance profile.
5. `origin.lambda.awsSecretAccessKey` : Your Secret Access Key.
7. `frontend.host` : The hostname the proxy should be available on. For Docker, this should always be `0.0.0.0`.
8. `frontend.port` : The port the proxy should bind to.
9. `frontend.endpoints` : The endpoints on which your app serves GraphQL. This defaults to `["/graphql"]`.

For full configuration details see [Proxy config](proxy-config.html).

<h3 id="run-the-proxy" title="Run the Proxy">2.3 Run the Proxy (Docker Container)</h3>

The Engine proxy is a docker image that you will deploy and manage separate from your server.

If you have a working [docker installation](https://docs.docker.com/engine/installation/), type the following lines in your shell (variables replaced with the correct values for your environment) to run the Engine proxy:

{% codeblock %}
engine_config_path=/path/to/engine.json
proxy_frontend_port=3001
docker run --env "ENGINE_CONFIG=$(cat "${engine_config_path}")" \
  -p "${proxy_frontend_port}:${proxy_frontend_port}" \
  gcr.io/mdg-public/engine:{% proxyDockerVersion %}
{% endcodeblock %}

This will make the Engine proxy available at `http://localhost:3001`.

It does not matter where you choose to deploy and manage your Engine proxy. We run our own on Amazon's [EC2 Container Service](https://aws.amazon.com/ecs/).

We recognize that almost every team using Engine has a slightly different deployment environment, and encourage you to <a href="mailto:support@apollographql.com">contact us</a> with feedback or for help if you encounter problems running the Engine proxy.

<h2 id="view-metrics-in-engine" title="View Metrics in Engine">3. View Metrics in Engine</h2>

Once your server is set up, navigate your new Engine service on https://engine.apollographql.com. Start sending requests to your Node server to start seeing performance metrics!
