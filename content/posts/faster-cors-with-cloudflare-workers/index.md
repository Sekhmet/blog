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

<div style="max-width:640px;margin:2em auto;font-family:system-ui,sans-serif;font-size:clamp(0.75rem,2.5vw,0.9rem)">
  <div style="display:flex;gap:4%;text-align:center;margin-bottom:0.75em">
    <div style="flex:1;font-weight:700;padding:0.5em 0.25em;border:2px solid currentColor;border-radius:6px;display:flex;align-items:center;justify-content:center">Browser</div>
    <div style="flex:1;font-weight:700;padding:0.5em 0.25em;border:2px solid currentColor;border-radius:6px;display:flex;align-items:center;justify-content:center">Cloudflare Worker</div>
    <div style="flex:1;font-weight:700;padding:0.5em 0.25em;border:2px solid currentColor;border-radius:6px;display:flex;align-items:center;justify-content:center">API Server</div>
  </div>
  <svg viewBox="0 0 640 300" style="width:100%;height:auto" aria-label="Sequence diagram showing OPTIONS requests handled by Cloudflare Worker while GET and POST go directly to API Server">
    <defs>
      <marker id="arr" markerWidth="8" markerHeight="6" refX="6" refY="3" orient="auto"><path d="M0,0 L8,3 L0,6" fill="currentColor"/></marker>
    </defs>
    <style>text{fill:currentColor;font-family:system-ui,sans-serif;font-size:14px}line{stroke:currentColor;stroke-width:1.5}</style>
    <!-- Lifelines -->
    <line x1="98" y1="0" x2="98" y2="300" stroke-dasharray="6,4" opacity=".15"/>
    <line x1="320" y1="0" x2="320" y2="300" stroke-dasharray="6,4" opacity=".15"/>
    <line x1="542" y1="0" x2="542" y2="300" stroke-dasharray="6,4" opacity=".15"/>
    <!-- OPTIONS Request: Browser -> Worker -->
    <text x="209" y="28" text-anchor="middle" font-weight="600">OPTIONS Request</text>
    <line x1="98" y1="36" x2="320" y2="36" marker-end="url(#arr)"/>
    <!-- 204 Response: Worker -> Browser -->
    <text x="209" y="68" text-anchor="middle" opacity=".7">204 Response</text>
    <line x1="320" y1="76" x2="98" y2="76" stroke-dasharray="8,4" marker-end="url(#arr)"/>
    <!-- GET Request: Browser -> API -->
    <text x="320" y="130" text-anchor="middle" font-weight="600">GET Request</text>
    <line x1="98" y1="138" x2="542" y2="138" marker-end="url(#arr)"/>
    <!-- 200 Response: API -> Browser -->
    <text x="320" y="170" text-anchor="middle" opacity=".7">200 Response</text>
    <line x1="542" y1="178" x2="98" y2="178" stroke-dasharray="8,4" marker-end="url(#arr)"/>
    <!-- POST Request: Browser -> API -->
    <text x="320" y="232" text-anchor="middle" font-weight="600">POST Request</text>
    <line x1="98" y1="240" x2="542" y2="240" marker-end="url(#arr)"/>
    <!-- 200 Response: API -> Browser -->
    <text x="320" y="272" text-anchor="middle" opacity=".7">200 Response</text>
    <line x1="542" y1="280" x2="98" y2="280" stroke-dasharray="8,4" marker-end="url(#arr)"/>
  </svg>
</div>

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
  "Access-Control-Allow-Methods": "GET,HEAD,PUT,PATCH,POST,DELETE",
  "Access-Control-Allow-Headers": "*",
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

1. Go to **Workers Routes**.
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
