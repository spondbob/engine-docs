---
title: Feature configuration
---

> These configuration guides are intended for use with the Engine Proxy, which is best used with non-JavaScript GraphQL server deployments.  For JavaScript servers, we recommend the use of [Apollo Server 2](/docs/apollo-server/v2/), which greatly reduces the necessary configuration.

<h2 id="automatic-persisted-queries">Automatic Persisted Queries (APQ)</h2>

Automatic persisted queries (APQs) is a performance technique which allows a hash to be sent to the server instead of the entire GraphQL query string.

> Apollo Server 2 reduces the setup necessary to use automatic persisted queries, and these instructions are only necessary when using the Apollo Engine Proxy.  To find out more about using APQ in Apollo Server 2, see the details in the [APQ guide](/docs/guides/performance.html#apq).

Inside Apollo Engine, the query registry is stored in a user-configurable cache.  Just like with response caching, this can either be an in-memory store (configured by default to be 50MB) within each Engine Proxy instance, or an external, configurable [memcached](https://memcached.org/) store.  Read more here about [how to configure caching](caching.html) in Engine.

To use automatic persisted queries (APQ) with Apollo Engine Proxy:

* Use Apollo Engine Proxy v1.0.1 or newer.
* If the GraphQL server is hosted on a different origin domain from where it will be accessed, setup the appropriate [CORS headers](https://en.wikipedia.org/wiki/Cross-origin_resource_sharing) using the `overrideGraphqlResponseHeaders` object on the frontend configuration:

  ```javascript
  frontends: [{
    overrideGraphqlResponseHeaders: {
      'Access-Control-Allow-Origin': '*',
    },
  }],
  ```

* Configure the client to use APQ using [these instructions](/docs/guides/performance.html#automatic-persisted-queried).
* Verify APQ is working properly using the [verification procedure](/docs/guides/performance.html#verify).
* Read [how it works](/docs/guides/performance.html#how-it-works) for additional details.

<h2 id="caching">Caching</h2>

To bring caching to GraphQL, we've developed [Apollo Cache Control](https://github.com/apollographql/apollo-cache-control), an open standard that allows servers to specify exactly what parts of a response can be cached and for how long.

Caching in Engine accepts cache control hints in a fine-grained way, but caches the entire result. Engine computes a cache privacy and expiration date by combining the data from all of the fields returned by the server for a particular request. It errs on the safe side, so shorter `maxAge` results override longer, and `PRIVATE` scope overrides `PUBLIC`.

<h4 id="why-cache">Why Cache?</h4>

API caching is a standard best practice, both to reduce the load on your servers, and to accelerate API responses and decrease page render times. But because GraphQL requests are often sent to a single endpoint with POST requests, existing HTTP caching solutions don’t work well for GraphQL APIs.

Apollo Engine Proxy reads Apollo Cache Control extensions, caching whole query responses based on a computed cacheability of each new query. Engine also includes an intuitive way to see how cache policies impact each query type, making it easy to:

* optimize your queries for cacheability
* refine field-level cache policies
* pinpoint slow components that could benefit from caching.

<h3 id="get-started">Get Started</h3>

There are just a few steps to enable response caching in Engine Proxy, and one of them is optional!

1. Enable cache control in the Apollo Server options.
1. Annotate your schema and/or resolvers with cache control hints.
1. _Optional:_ Configure cache options in your Engine Proxy configuration.

<h3 id="enable-cache-control">1. Apollo Server: cacheControl</h3>

If you're using Apollo Server for your Node GraphQL server, the only server code change required is to add `tracing: true` and `cacheControl: true` to the options passed to the Apollo Server middleware function for your framework of choice. For example, for Express:

```js
app.use('/graphql', bodyParser.json(), graphqlExpress({
  schema,
  context: {},
  tracing: true,
  cacheControl: true
}));
```

Apollo Server includes built-in support for Apollo Cache Control from version 1.2.0 onwards. If you are using `express-graphql`, we recommend you switch to Apollo Server to use caching. Both `express-graphql` and Apollo Server are based on the [`graphql-js`](https://github.com/graphql/graphql-js) reference implementation, and switching should only require changing a few lines of code. We're working with the community to add support for Apollo Cache Control to non-Node GraphQL server libraries. <a href="mailto:support@apollographql.com">Contact us</a> if you are interested in joining the community to work on support for `express-graphql` or non-Node GraphQL servers.

Next, set [hints in your schema](#hints-to-schema), or [dynamically in your resolvers](#resolver-hints).

<h3 id="cache-hints" name="cacheHints">2. Add Cache Hints</h3>

There are two ways to add cache hints to your application --- either dynamically on your resolvers, or statically on your schema types and fields. Each cacheControl hint has two parameters.

* The `maxAge` parameter defines the number of seconds that Engine Proxy should serve the cached response.
* The `scope` parameter declares that a unique response should be cached for every user (`PRIVATE`) or a single response should be cached for all users (`PUBLIC`/default).

<h4 id="max-age-combination">maxAge</h4>

To determine the expiration time of a particular query, Engine looks at all of the `maxAge` hints returned by the server, and picks the shortest. For example, the above result indicates a `240` maxAge for one field, and `30` for another, which means that Engine will use `30` as the overall expiration time for the whole result.

The biggest benefit of collecting the cache hints on a per-field basis is that you can use the Engine UI to understand your cache hit rates and the overall `maxAge` for an operation. Here's what the caching view looks like:

![Cache trace](../img/cache-trace.png)
**Note** For example, if your query calls a type with a field referencing list of type objects, such as `[Post]` referencing `Author` in the `author` field, Engine will consider the `maxAge` of the `Author` type as well. 

<h4 id="public-vs-private">Public vs. private scope</h4>

Apollo Engine supports caching of personalized responses using the `scope: PRIVATE` cache hint. Private caching requires that Engine identify unique users, using the methods defined in the `sessionAuth` configuration section.

Engine supports extracting users' identity from an HTTP header (specified in `header`), or an HTTP cookie (specified in `cookie`).

For security, Engine can be configured to verify the extracted identity before serving a cached response. This allows your service to verify the session is still valid and avoid replay attacks.
This verification is performed by HTTP request, to the URL specified in `tokenAuthUrl`.

The token auth URL will receive an HTTP POST containing: `{"token": "AUTHENTICATION-TOKEN"}`.
It should return an HTTP `200` response if the token is still considered valid.
It may optionally return a JSON body:
* `{"ttl": 300}` to indicate the session token check can be cached for 300 seconds.
* `{"id": "alice"}` to indicate an internal user ID that should be used for identification. By returning a persistent identifier such as a database key, Engine's cache can follow a user across sessions and devices.
* `{"ttl": 600, "id": "bob"}` to combine both.

Authentication checks with `ttl>0` will be cached in a `store` named in `sessionAuth`, or in the default 50MB in-memory store.

<h3 id="hints-to-schema" name="schemaHints">Cache Hints in the Schema</h3>

Cache hints can be added to your schema using directives on your types and fields. When executing your query, these hints will be added to the response and interpreted by Engine to compute a cache policy for the response. 

Engine sets cache TTL as the lowest `maxAge` in the query path.

```graphql
type Post @cacheControl(maxAge: 240) {
  id: Int!
  title: String
  author: Author
  votes: Int @cacheControl(maxAge: 500)
  readByCurrentUser: Boolean! @cacheControl(scope: PRIVATE)
}

type Author @cacheControl(maxAge: 60) {
  id: Int
  firstName: String
  lastName: String
  posts: [Post]
}
```

You should receive cache control data in the `extensions` field of your response:

```js
"cacheControl": {
  "version": 1,
  "hints": [
    {
      "path": [
        "post"
      ],
      "maxAge": 240
    },
    {
      "path": [
        "post",
        "votes"
      ],
      "maxAge": 30
    },
    {
      "path": [
        "post",
        "readByCurrentUser"
      ],
      "scope": "PRIVATE"
    }
  ]
}
```

For the above schema, there are a few ways to generate different TTLs depending on your query. Take the following examples:

*Example 1*
```graphql
query getPostsForAuthor { 
    Author { 
      posts 
    } 
  }
```

`getPostsForAuthor` will have `maxAge` of 60 seconds, even though the `Post` object has `maxAge` of 240 seconds.

*Example 2*
```graphql
query getTitleForPost { 
  Post { 
    title 
  } 
}
```

`getTitleForPost` will have `maxAge` of 240 seconds (inherited from Post), even though the `title` field has no `maxAge` specified.

*Example 3* 
```graphql
query getVotesForPost { 
  Post { 
    votes 
    } 
  }
```

`getVotesForPost` will have `maxAge` of 240 seconds, even though the `votes` field has a higher `maxAge`.

<h4 id="resolver-hints" name="resolverHints">Dynamic Cache Hints in the Resolvers</h4>
If you'd like to add cache hints dynamically, you can use a programmatic API from within your resolvers.

```js
const resolvers = {
  Query: {
    post: (_, { id }, _, { cacheControl }) => {
      cacheControl.setCacheHint({ maxAge: 60 });
      return find(posts, { id });
    }
  }
}
```

<h4 id="default-max-age">Setting a default `maxAge`</h4>

The power of cache hints comes from being able to set them precisely to different values on different types and fields based on your understanding of your implementation's semantics. But when getting started with Apollo Cache Control, you might just want to apply the same `maxAge` to most of your resolvers. You can specify a default max age when you set up `cacheControl` in your server. This max age will be applied to all resolvers which don't explicitly set `maxAge` via schema hints (including schema hints on the type that they return) or the programmatic API. You can override this for a particular resolver or type by setting `@cacheControl(maxAge: 0)`.

Just like when you set `@cacheControl(maxAge: 5)` explicitly on a field or a type, data is considered to be public by default and the cache will be shared among all users of your site, so when using this option, be sure that you're really OK with creating a shared cache for all of your GraphQL queries. You can still override a specific type or resolver to use the private cache by setting `@cacheControl(scope: PRIVATE)`.

For example, for Express:

```javascript
app.use('/graphql', bodyParser.json(), graphqlExpress({
  schema,
  context: {},
  tracing: true,
  cacheControl: {
    defaultMaxAge: 5,
  },
}));
```

Setting `defaultMaxAge` requires `apollo-server-*` 1.3.4 or newer.


<h3 id="engine-cache-config">3. Optional: Configure cache options</h3>

As long as you're using Engine 1.0 or newer, you don't have to configure anything in your Engine configuration to use public response caching.  Engine provides a default 50MB in-memory cache.

To enable private response caching or to configure details of how caching works, there are a few fields in the Engine configuration (ie, argument to `new ApolloServer`) that are relevant.

Here is an example of changing the Engine config for caching `scope: PUBLIC` responses to use memcached instead of an in-memory cache.
Since no `privateFullQueryStore` is provided, `scope: PRIVATE` responses will not be cached.

```js
const engine = new ApolloEngine({
  stores: [{
    memcache: {
      url: ['localhost:4567'],
    },
  }],
  // ...
});
```

Below is an example of an Engine config for caching `scope: PUBLIC` and `scope: PRIVATE` responses, using the default (empty-string-named 50MB in-memory cache) for public responses and authorization tokens, and memcached for private responses.
By using a private response cache, we guarantee that a response affecting multiple users is never evicted for a response affecting only a single user.

```js
const engine = new ApolloEngine({
  stores: [{
    name: 'privateResponseMemcache',
    memcache: {
      url: ['localhost:4567'],
    },
  }],
  sessionAuth: {
    header: 'Authorization',
    tokenAuthUrl: 'https://auth.mycompany.com/engine-auth-check',
  },
  queryCache: {
    privateFullQueryStore: 'privateResponseMemcache',
    // By not mentioning publicFullQueryStore, we keep it enabled with
    // the default empty-string-named in-memory store.
  },
  // ...
});
```


Here's an explanation of these config fields:

<h4 id="config.stores">stores</h4>

Stores is an array of places for Engine to store data such as: query responses, authentication checks, or persisted queries.

Every store must have a unique `name`. The empty string is a valid name; there is a default in-memory 50MB cache with the empty string for its name which is used for any caching feature if you don't specify a store name.  You can specify the name of `"disabled"` to any caching feature to turn off that feature.

Engine supports two types of stores:

* `inMemory` stores provide a bounded LRU cache embedded within the Engine Proxy.
  Since there's no external servers to configure, in-memory stores are the easiest to get started with.
  Since there's no network overhead, in-memory stores are the fastest option.
  However, if you're running multiple copies of Engine Proxy, their in-memory stores won't be shared --- a cache hit on one server may be a cache miss on another server.
  In memory caches are wiped whenever Engine Proxy restarts.

  The only configuration required for in memory stores is `cacheSize` --- an upper limit specified in bytes. It defaults to 50MB.

* `memcache` stores use external [Memcached](https://memcached.org/) server(s) for persistence.
  This provides a shared location for multiple copies of Engine Proxy to achieve the same cache hit rate.
  This location is also not wiped across Engine Proxy restarts.

  Memcache store configuration requires an array of addresses called `url`, for the memcached servers. (This name is misleading: the values are `host:port` without any URL scheme like `http://`.) All addresses must contain both host and port, even if using the default memcached port. The AWS Elasticache discovery protocol is not currently supported.
  `keyPrefix` may also be specified, to allow multiple environments to share a memcached server (i.e. dev/staging/production).

We suggest developers start with an in-memory store, then upgrade to Memcached if the added deployment complexity is worth it for production.
This will give you much more control over memory usage and enable sharing the cache across multiple Engine proxy instances.

<h4 id="config.sessionAuth">sessionAuth</h4>

This is useful when you want to do per-session response caching with Engine. To be able to cache results for a particular user, Engine needs to know how to identify a logged-in user. In this example, we've configured it to look for an `Authorization` header, so private data will be stored with a key that's specific to the value of that header.

You can specify that the session ID is defined by either a header or a cookie. Optionally, you can specify a REST endpoint which the Engine Proxy can use to determine whether a given token is valid.

<h4 id="config.queryCache">queryCache</h4>

This maps the types of result caching Engine performs to the stores you've defined in the `stores` field.
In this case, we're sending public and private cached data to unique stores, so that responses affecting multiple users will never be evicted for responses affecting a single user.

If you leave `queryCache.publicFullQueryStore` blank, it will use the default 50MB in-memory cache. Set it to `"disabled"` to turn off the cache.

If you configure `sessionAuth` but leave `queryCache.privateFullQueryStore` blank, it will use the default 50MB in-memory cache. Set it to `"disabled"` to turn off the cache.

<h3 id="visualizing">Visualizing caching</h3>

One of the best parts about caching with Engine is that you can easily see how it's working once you set it up. It deeply integrates into the [performance tracing](../features/performance.html) views, so that you can understand how caching is helping you decrease your server response times. Here are what the charts in the performance view look like when you've successfully enabled caching:

![Cache rate on volume chart](../img/cache-rate.png)

The volume chart now shows how many of your requests hit the cache instead of the underlying server.

![Cache rate on heat map](../img/cache-histogram.png)

The histogram uses differently-colored bars to represent cache vs. non-cache requests. So you can easily see that the cached requests are much much faster, with Engine responding to those requests in microseconds rather than the 50-100 milliseconds it would take to hit the underlying server.


<h3 id="http-headers">How HTTP headers affect caching</h3>

The main way that your GraphQL server specifies cache behavior is through the `cacheControl` GraphQL extension, which is rendered in the body of a GraphQL response. However, Engine also understands and sets several caching-related HTTP headers.

#### HTTP headers interpreted by Engine

Engine will never decide to cache responses in its response cache unless you tell it to with the `cacheControl` GraphQL extension. However, Engine does observe some HTTP headers and can use them to restrict caching further than what the extension says.  These headers include:

* `Cache-Control` **response** header: If the `Cache-Control` response header contains `no-store`, `no-cache`, or `private`, Engine will not cache the response. If the `Cache-Control` response header contains `max-age` or `s-maxage` directives, then Engine will not cache any data for longer than the specified amount of time. (That is, data will be cached for the minimum of the header-provided `max-age` and the extension-provided `maxAge`.)  `s-maxage` takes precedence over `max-age`.
* `Cache-Control` **request** header: If the `Cache-Control` request header contains `no-cache`, Engine will not look in the cache for responses. If the `Cache-Control` request header contains `no-store`, Engine will not cache the response.
* `Expires` response header: If the `Expires` response header is present, then Engine will not cache any data past the given date.  The `Cache-Control` directives `s-maxage` and `max-age` take precedence over `Expires`.
* `Vary` response header: If the `Vary` response header is present, then Engine will not return this response to any request whose headers named in the `Vary` header don't match the request that created this response. (For example, if a request had a `Accept-Language: de` header and the response had a `Vary: Accept-Language` header, then that response won't be returned from the cache to any response that does not also have a `Accept-Language: de` header.)  Additionally, Engine uses a heuristic to store requests that have different values for headers that it suspects may show up in the response `Vary` header under different cache keys; currently that heuristic is that it assumes that any header that has ever shown up in a `Vary` header in a GraphQL response may be relevant.

#### HTTP headers set by Engine

When returning a GraphQL response which is eligible for the full-query cache (ie, all of the data has a non-zero `maxAge` set in the `cacheControl` GraphQL extension), Engine sets the `Cache-Control` header with a `max-age` directive equal to the minimum `maxAge` of all data in the response. If any of the data in the response has a `scope: PRIVATE` hint, the `Cache-Control` header will include the `private` directive; otherwise it will include the `public` directive. This header completely replaces any `Cache-Control` and `Expires` headers provided by your GraphQL server.

<h2 id="cdn">CDN integration</h2>

Many high-traffic web services use content delivery networks (CDNs) such as [Cloudflare](https://www.cloudflare.com/), [Akamai](https://www.akamai.com/) or [Fastly](https://www.fastly.com/) to cache their content as close to their clients as possible.

> Apollo Server 2 supports CDN integration out of the box and doesn't require the Engine Proxy. To learn how, read through the [guide on CDN integration](/docs/guides/performance.html#cdn). For other server implementations, the Engine Proxy makes it straightforward to use CDNs with GraphQL queries whose responses can be cached while still passing more dynamic queries through to your GraphQL server.

To use Engine Proxy behind a CDN, you need to be able to tell the CDN which GraphQL responses it's allowed to cache, and you need to make sure that your GraphQL requests arrive in a format that CDNs cache. Engine Proxy supports this via its [caching](./caching.html) and [automatic persisted queries](./auto-persisted-queries.html) features. This page explains the basic steps for setting up these features to work with CDNs; for more details on how to configure these features, see their respective pages.

<h3 id="enable-cache-control" title="1. Enable cache control">Step 1: Enable cache control in your Apollo Server</h3>

Add `cacheControl: true` to your Apollo Server middleware invocation, such as `graphqlExpress`. See [the caching docs](./caching.html#enable-cache-control) for more details on enabling Cache Control. For Apollo Server 2, your call should look something like:

```js
const server = new ApolloServer({
  typeDefs,
  resolvers,
  tracing: true,
  cacheControl: true
  // We set `engine` to false, so that the new agent is not used.
  engine: false,
);
```

<h3 id="cache-hints" title="2. Add cache hints">Step 2: Add cache hints to your GraphQL schema</h3>

Add cache hints to your GraphQL schema so that Engine Proxy knows which fields and types are cacheable and for how long. For example, to say that all fields that return an `Author` should be cached for 60 seconds, and that the `posts` field should itself be cached for 180 seconds, write this in your schema:

```graphql
type Author @cacheControl(maxAge: 60) {
  id: Int
  firstName: String
  lastName: String
  posts: [Post] @cacheControl(maxAge: 180)
}
```

See [the cache hints docs](./caching.html#cache-hints) for more details on cache hints, including how to specify hints dynamically inside resolvers, how to set a default `maxAge` for all fields, and how to specify that a field should be cached for specific users only (in which case CDNs should ignore it).

Once you've done these two steps, Engine Proxy will serve the HTTP `Cache-Control` header on _fully cacheable_ responses, so that any CDN in front of Engine Proxy will know which responses can be cached and for how long! By _fully cacheable_, we mean any response containing only data with non-zero `maxAge`; the header will refer to the minimum `maxAge` value across the whole response, and it will be `public` unless some of the data is tagged `scope: PRIVATE`. You should be able to observe this header in your browser's dev tools. Engine Proxy will also cache the responses in its own default public in-memory cache.

<h3 id="enable-apq" title="3. Enable persisted queries">Step 3: Enable automatic persisted queries</h3>

At this point, GraphQL requests are still POST requests.  Most CDNs will only cache GET requests and GET requests generally work best if the URL is of a bounded size. To work with this, enable Apollo Engine Proxy's Automatic Persisted Queries (APQ) support. This allows clients to send short hashes instead of full queries, and you can configure it to use GET requests for those queries.

To do this, follow the steps in the [guide above](#automatic-persisted-queries).  After completing the steps in that section of the guide, you should be able to observe queries being sent as `GET` requests with the appropriate `Cache-Control` response headers using your browser's developer tools.

<h3 id="setup-cdn" title="4. Set up your CDN">Step 4: Set up your CDN!</h3>

How precisely this works relies upon which CDN you chose. Configure your CDN to send requests to your Engine Proxy-powered GraphQL app. For some CDNs, you may need to specially configure your CDN to honor origin Cache-Control headers; for example, here is [Akamai's documentation on that setting](https://learn.akamai.com/en-us/webhelp/ion/oca/GUID-57C31126-F745-4FFB-AA92-6A5AAC36A8DA.html). If all is well, your cacheable queries should now be cached by your CDN! Note that requests served directly by your CDN will not show up in your Engine dashboard.

<h2 id="tracing">Tracing</h2>

[Apollo Tracing](https://github.com/apollographql/apollo-tracing) is a GraphQL extension to expose performance tracing data as part of your GraphQL responses. Engine relies on receiving data in the Apollo Tracing format to create its performance telemetry reports.

There are currently implementations that allow you to use the Apollo Tracing format with the following GraphQL servers:

1. **Node** with [Apollo Server](https://www.apollographql.com/docs/apollo-server/): [apollo-tracing-js](https://github.com/apollographql/apollo-tracing-js). supports Express, Hapi, Koa, Restify, and Lambda.
2. **Ruby** with [GraphQL-Ruby](http://graphql-ruby.org/): [apollo-tracing-ruby](https://github.com/uniiverse/apollo-tracing-ruby)
3. **Java** with [GraphQL-Java](https://github.com/graphql-java/graphql-java): Built in! [Read the docs](http://graphql-java.readthedocs.io/en/latest/instrumentation.html#apollo-tracing-instrumentation)
4. **Scala** with [Sangria](https://github.com/sangria-graphql/sangria): [Use this snippet](https://gist.github.com/OlegIlyenko/124b55e58609ad45fcec276f15158d16)
5. **Elixir** with [Absinthe](https://github.com/absinthe-graphql/absinthe): [apollo-tracing-elixir](https://github.com/sikanhe/apollo-tracing-elixir)

Using a different server? <a href="mailto:support@apollographql.com">Let us know</a> – the development of our tracing agents is community driven and we would love to start a conversation with you!

<h2 id="query-batching">Query batching</h2>

Query batching allows your client to batch multiple queries into one request.  This means that if you render several view components within a short time interval, for example a navbar, sidebar, and content, and each of those do their own GraphQL query, the queries can be sent together in a single roundtrip.  

A batch of queries can be sent by simply sending a JSON-encoded array of queries in the request:

```js
[ 
 { "query": "{
  feed(limit: 2, type: NEW) {
    postedBy {
      login
    }
    repository {
      name
      owner {
        login
      }
    }
  }
}" }, 
 { "query": "query CurrentUserForLayout {
  currentUser {
    __typename
    avatar_url
    login
  }
}" }
] 
```

Batched requests to servers that don’t support batching fail without explicit code to handle batching, however Engine ships with batched request handling built-in.  

<h3 id="apollo-server-batch-support" title="Batching and Engine">Batching and Engine</h3>

If a batch of queries is sent, the batches are fractured by the Engine proxy and individual queries are sent to origins in parallel.  Engine will wait for all the responses to complete and send a single response back to the client.  The response will be an array of GraphQL results:

```js
[{
  "data": {
    "feed": [
      {
        "postedBy": {
          "login": "AleksandraKaminska"
        },
        "repository": {
          "name": "GitHubApp",
          "owner": {
            "login": "AleksandraKaminska"
          }
        }
      },
      {
        "postedBy": {
          "login": "ashokhein"
        },
        "repository": {
          "name": "memeryde",
          "owner": {
            "login": "ashokhein"
          }
        }
      }
    ]
  }
},
{
  "data": {
    "currentUser": {
      "__typename": "User",
      "avatar_url": "https://avatars2.githubusercontent.com/u/11861843?v=4",
      "login": "johannakate"
    }
  }
}]
```

<h3 id="apollo-server-batch-support" title="Batching, Apollo Client & Engine">Batching in Apollo Client with Engine</h3>

Apollo Client has built-in support for batching queries in your client application.  To learn how to use query batching with Apollo Client, visit the in-depth guide on our package [`apollo-link-batch-http`](https://www.apollographql.com/docs/link/links/batch-http.html).


<h3 id="apollo-server-batch-support" title="Batching, Apollo Server & Engine">Batching in Apollo Server with Engine</h3>

If your origin supports batching and you'd like to pass entire batches through, set `"supportsBatch": true` within the origins section of the configuration:

```js
const engine = new ApolloEngine({
  apiKey: "ENGINE_API_KEY",
  origins: [{
    supportsBatch: true,
  }],
});
```

Add this setting, and you're good to go!

If you have questions, we're always available in the public [Apollo Slack #engine channel](https://www.apollographql.com/slack).


