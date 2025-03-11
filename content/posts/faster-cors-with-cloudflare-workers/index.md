+++
date = '2025-03-06T13:32:19+01:00'
draft = false
title = 'Speeding up CORS preflight requests with Cloudflare Workers'
description = 'Using Cloudflare Workers to handle all CORS preflight requests to make them as fast as possible.'
+++

As a security measure browsers perform [CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)
preflight requests, but sometimes those preflight requests add significant latency -
for example if your CORS-enabled API is hosted in single location and you have users all across the world.
Some users located far away from your API server might need to wait extra 200-300ms to just get CORS response.

This can be improved by routing all `OPTIONS` requests to Cloudflare Worker that should respond
quickly no matter where your users are.

In my case it didn't make sense to route all API traffic through Cloudflare worker (which
can be done using its `fetch` method). With suggested solution only `OPTIONS` requests
are handled by the worker, other traffic is handled directly by the API.

Following diagram shows what we are trying to achieve:

```goat
┌──────────────────────┐      ┌──────────────────────┐      ┌──────────────────────┐
│       Browser        │      │  Cloudflare Worker   │      │      API Server      │
└──────────────────────┘      └──────────────────────┘      └──────────────────────┘
            │                             │                             │
            │                             │                             │
            │                             │                             │
            │       OPTIONS Request       │                             │
            ├────────────────────────────▶│                             │
            │                             │                             │
            │                             │                             │
            │        204 Response         │                             │
            │◀─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┤                             │
            │                             │                             │
            │                                                           │
            │                                                           │
            │                                                           │
            │                                                           │
            │                       GET Request                         │
            ├──────────────────────────────────────────────────────────▶│
            │                                                           │
            │                                                           │
            │                      200 Response                         │
            │◀─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┤
            │                                                           │
            │                                                           │
            │                                                           │
            │                                                           │
            │                                                           │
            │                      POST Request                         │
            ├──────────────────────────────────────────────────────────▶│
            │                                                           │
            │                                                           │
            │                      200 Response                         │
            │◀─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┤
            │                                                           │
```

With this setup we get benefit of faster CORS requests, but at the same time we are not
adding extra latency by proxying every request.

Cloudflare doesn't have direct way to connect worker just for specfiic HTTP method,
so we need to workaround it using page rules.

## Step 1. Create worker

First you need to create Cloudflare Worker. It doesn't need to have custom domain as
we won't be using it directly.

This is simplest version that just returns `Access-Control-Allow-Origin` header.

```js
const corsHeaders = {
  "Access-Control-Allow-Origin": "*",
};

export default {
  async fetch(request, env, ctx) {
    if (request.method === "OPTIONS") {
      return new Response(null, {
        status: 204,
        headers: corsHeaders,
      });
    }

    return new Response("Proxy is working. Will respond to OPTIONS");
  },
};
```

To create worker in Cloudflare account dashboard:

1. Go to **Compute (Workers)**.
2. Click **Create**.
3. Select _Hello World_ template.
4. Pick some worker name and click **Deploy**.
5. Click **Edit code**.
6. Use worker code from above and click **Deploy**.

## Step 2. Create rule to redirect `OPTIONS` requests to `/cors-proxy`

We will redirect all `OPTIONS` requests to some path on your domain. Exact path isn't
really important as we will just use it to connect worker in later steps.

In Cloudflare dashboard for you domain:

1. Go to **Rules**.
2. Click **Create rule**.
3. Select **URL Rewrite Rule**.
4. Use following rule details:
   1. **Rule name**: Rewrite OPTIONS request to /cors-proxy
   2. **If incoming requests match…**: Custom filter expression
   3. Use two fields with AND operator:
      - Hostname - equals - api.example.com
      - Request Method - equals - OPTIONS
   4. **Then...**:
      1. Path - rewrite to - static - `/cors-proxy`
      2. Query - preserve
5. Click **Save**.

With this change all `OPTIONS` requests will be forwarded to `api.example.com/cors-proxy`.
This isn't really useful on itself, but with this rule we can now connect our worker
to this path.

## Step 3. Create Worker Route to connect your worker

Again, in Cloudflare dashboard for your domain:

1. Go to **Worker Route**.
2. Click **Add route**.
3. Fill it in as:
   - **Route**: `api.example.com/cors-proxy`
   - **Worker**: Select worker you created in step 1.
4. Click **Save**.

## Step 4. Enjoy faster CORS requests

You can now enjoy the results. Because preflight requests don't ever hit your actual API
and are served by worker you will get good response times for those requests regardless
where you access it from as they run on Cloudflare's edge network.

Before:
![Preflight response times without worker](direct.png)

After:
![Preflight response times with worker](worker.png)
