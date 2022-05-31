# Upstash Redis

`@upstash/redis` is an HTTP/REST based Redis client for typescript, built on top
of [Upstash REST API](https://docs.upstash.com/features/restapi).

[![Tests](https://github.com/upstash/upstash-redis/actions/workflows/tests.yaml/badge.svg)](https://github.com/upstash/upstash-redis/actions/workflows/tests.yaml)
![npm (scoped)](https://img.shields.io/npm/v/@upstash/redis)
![npm bundle size](https://img.shields.io/bundlephobia/minzip/@upstash/redis)

It is the only connectionless (HTTP based) Redis client and designed for:

- Serverless functions (AWS Lambda ...)
- Cloudflare Workers (see
  [the example](https://github.com/upstash/upstash-redis/tree/main/examples/cloudflare-workers))
- Fastly Compute@Edge (see
  [the example](https://github.com/upstash/upstash-redis/tree/main/examples/fastly))
- Next.js, Jamstack ...
- Client side web/mobile applications
- WebAssembly
- and other environments where HTTP is preferred over TCP.

See
[the list of APIs](https://docs.upstash.com/features/restapi#rest---redis-api-compatibility)
supported.

## Upgrading to v1.4.0 **(ReferenceError: fetch is not defined)**

If you are running on nodejs v17 and earlier, `fetch` will not be natively
supported. Platforms like Vercel, Netlify, Deno, Fastly etc. provide a polyfill
for you. But if you are running on bare node, you need to either specify a
polyfill yourself or change the import path to:

```typescript
import { Redis } from "@upstash/redis/with-fetch";
```

## Upgrading from v0.2.0?

Please read the
[migration guide](https://github.com/upstash/upstash-redis#migrating-to-v1). For
further explanation we wrote a
[blog post](https://blog.upstash.com/upstash-redis-sdk-v1).

## Quick Start

### Install

#### npm

```bash
npm install @upstash/redis
```

#### Deno

```ts
import { Redis } from "https://deno.land/x/upstash_redis/mod.ts";
```

### Create database

Create a new redis database on [upstash](https://console.upstash.com/)

### Environments

We support various platforms, such as nodejs, cloudflare and fastly. Platforms
differ slightly when it comes to environment variables and their `fetch` api.
Please use the correct import when deploying to special platforms.

#### Node.js

Examples: Vercel, Netlify, AWS Lambda

If you are running on nodejs you can set `UPSTASH_REDIS_REST_URL` and
`UPSTASH_REDIS_REST_TOKEN` as environment variable and create a redis instance
like this:

```ts
import { Redis } from "@upstash/redis"

const redis = new Redis({
  url: <UPSTASH_REDIS_REST_URL>,
  token: <UPSTASH_REDIS_REST_TOKEN>,
})

// or load directly from env
const redis = Redis.fromEnv()
```

If you are running on nodejs v17 and earlier, `fetch` will not be natively
supported. Platforms like Vercel, Netlify, Deno, Fastly etc. provide a polyfill
for you. But if you are running on bare node, you need to either specify a
polyfill yourself or change the import path to:

```typescript
import { Redis } from "@upstash/redis/with-fetch";
```

- [Code example](https://github.com/upstash/upstash-redis/blob/main/examples/nodejs)

#### Cloudflare Workers

Cloudflare handles environment variables differently than nodejs. Please add
`UPSTASH_REDIS_REST_URL` and `UPSTASH_REDIS_REST_TOKEN` using
`wrangler secret put ...` or in the cloudflare dashboard.

Afterwards you can create a redis instance:

```ts
import { Redis } from "@upstash/redis/cloudflare"

const redis = new Redis({
  url: <UPSTASH_REDIS_REST_URL>,
  token: <UPSTASH_REDIS_REST_TOKEN>,
})


// or load directly from global env

// service worker
const redis = Redis.fromEnv()


// module worker
export default {
  async fetch(request: Request, env: Bindings) {
    const redis = Redis.fromEnv(env)
    // ...
  }
}
```

- [Code example Wrangler 2](https://github.com/upstash/upstash-redis/tree/main/examples/cloudflare-workers)
- [Code example Wrangler 1](https://github.com/upstash/upstash-redis/tree/main/examples/cloudflare-workers-with-wrangler-1)
- [Documentation](https://docs.upstash.com/redis/tutorials/cloudflare_workers_with_redis)

#### Fastly

Fastly introduces a concept called
[backend](https://developer.fastly.com/reference/api/services/backend/). You
need to configure a backend in your `fastly.toml`. An example can be found
[here](https://github.com/upstash/upstash-redis/blob/main/examples/fastly/fastly.toml).
Until the fastly api stabilizes we recommend creating an instance manually:

```ts
import { Redis } from "@upstash/redis/fastly"

const redis = new Redis({
  url: <UPSTASH_REDIS_REST_URL>,
  token: <UPSTASH_REDIS_REST_TOKEN>,
  backend: <BACKEND_NAME>,
})
```

- [Code example](https://github.com/upstash/upstash-redis/tree/main/examples/fastly)
- [Documentation](https://blog.upstash.com/fastly-compute-edge-with-redi)

#### Deno

Examples: [Deno Deploy](https://deno.com/deploy),
[Netlify Edge](https://www.netlify.com/products/edge/)

```ts
import { Redis } from "https://deno.land/x/upstash_redis/mod.ts"

const redis = new Redis({
  url: <UPSTASH_REDIS_REST_URL>,
  token: <UPSTASH_REDIS_REST_TOKEN>,
})

// or
const redis = Redis.fromEnv();
```

### Working with types

Most commands allow you to provide a type to make working with typescript
easier.

```ts
const data = await redis.get<MyCustomType>("key");
// data is typed as `MyCustomType`
```

## Migrating to v1

### Explicit authentication

The library is no longer automatically trying to load connection secrets from
environment variables. You must either supply them yourself:

```ts
import { Redis } from "@upstash/redis"

const redis = new Redis({
  url: <UPSTASH_REDIS_REST_URL>,
  token: <UPSTASH_REDIS_REST_TOKEN>,
})
```

Or use one of the static constructors to load from environment variables:

```ts
// Nodejs
import { Redis } from "@upstash/redis";
const redis = Redis.fromEnv();
```

```ts
// or when deploying to cloudflare workers
import { Redis } from "@upstash/redis/cloudflare";
const redis = Redis.fromEnv();
```

### Error handling

Errors are now thrown automatically instead of being returned to you.

```ts
// old
const { data, error } = await set("key", "value");
if (error) {
  throw new Error(error);
}

// new
const data = await redis.set("key", "value"); // error is thrown automatically
```

## Pipeline

`v1.0.0` introduces redis pipelines. Pipelining commands allows you to send a
single http request with multiple commands.

```ts
import { Redis } from "@upstash/redis";

const redis = new Redis({
  /* auth */
});

const p = redis.pipeline();

// Now you can chain multiple commands to create your pipeline:

p.set("key", 2);
p.incr("key");

// or inline:
p.hset("key2", "field", { hello: "world" }).hvals("key2");

// Execute the pipeline once you are done building it:
// `exec` returns an array where each element represents the response of a command in the pipeline.
// You can optionally provide a type like this to get a typed response.
const res = await p.exec<[Type1, Type2, Type3]>();
```

For more information about pipelines using REST see
[here](https://blog.upstash.com/pipeline).

### Advanced

A low level `Command` class can be imported from `@upstash/redis` in case you
need more control about types and or (de)serialization.

By default all objects you are storing in redis are serialized using
`JSON.stringify` and recursively deserialized as well. Here's an example how you
could customize that behaviour. Keep in mind that you need to provide a `fetch`
polyfill if you are running on nodejs. We recommend
[isomorphic-fetch](https://www.npmjs.com/package/isomorphic-fetch).

```ts
import { Command } from "@upstash/redis/commands"
import { HttpClient } from "@upstash/redis/http"

/**
 * TData represents what the user will enter or receive,
 * TResult is the raw data returned from upstash, which may need to be
 * transformed or parsed.
 */
const deserialize: (raw: TResult) => TData = ...

class CustomGetCommand<TData, TResult> extends Command<TData | null, TResult | null> {
  constructor(key: string, ) {
    super(["get", key], { deserialize })
  }
}

const client = new HttpClient({
  baseUrl: <UPSTASH_REDIS_REST_URL>,
  headers: {
    authorization: `Bearer ${<UPSTASH_REDIS_REST_TOKEN>}`,
  },
})

const res = new CustomGetCommand("key").exec(client)
```

### Additional information

#### `keepalive`

`@upstash/redis` is capable of reusing connections where possible to minimize
latency. Connections can be reused if the client is stored in memory and not
initialized with every new function invocation. The easiest way to achieve this
is by creating the client outside of your handler and adding an https agent:

```ts
// Nextjs api route
import { Redis } from "@upstash/redis";
import https from "https";

const redis = Redis.fromEnv({
  agent: new https.Agent({ keepAlive: true }),
});

export default async function (req, res) {
  // use redis here
}
```

Whenever your hot lambda receives a new request the client is already
initialized and the previously established connection to upstash is reused.

#### Javascript MAX_SAFE_INTEGER

Javascript can not handle numbers larger than `2^53 -1` safely and would return
wrong results when trying to deserialize them. In these cases the default
deserializer will return them as string instead. This might cause a mismatch
with your custom types.

```ts
await redis.set("key", "101600000000150081467");
const res = await redis<number>("get");
```

In this example `res` will still be a string despite the type annotation. Please
keep that in mind and adjust accordingly.

## Docs

See [the documentation](https://docs.upstash.com/features/javascriptsdk) for
details.

## Contributing

### [Install Deno](https://deno.land/#installation)

### Database

Create a new redis database on [upstash](https://console.upstash.com/) and copy
the url and token

### Running tests

```sh
UPSTASH_REDIS_REST_URL=".." UPSTASH_REDIS_REST_TOKEN=".." deno test -A
```

```
```
