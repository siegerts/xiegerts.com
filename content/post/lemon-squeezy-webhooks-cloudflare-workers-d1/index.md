+++
title = "Verifying Lemon Squeezy Subscription Webhooks in Cloudflare Workers"
description = "Using Cloudflare Workers, D1, and Drizzle ORM to verify and save Lemon Squeezy subscription webhook data."

tags = [
  "hono",
  "lemon-squeezy",
	"serverless",
  "cloudflare",
  "wrangler",
  "sqlite",
  "d1",
	"web crypto api",
	"subscription payments"
]

categories = ["Serverless"]

date = 2024-06-27T10:33:55-05:00

+++

I've been using Lemon Squeezy to handle [subscription billing](https://docs.lemonsqueezy.com/help/products/subscriptions) in several projects. Lemon Squeezy offers an easy way to manage subscriptions and payments, similar to the DX of Stripe, but includes tax remittance and compliance as a Merchant of Record (MoR). So, using Lemon Squeezy helps offload the complexities of tax, compliance, and dispute handling. Additionally, I like the idea of having the built-in email marketing and affiliate portal as optional features.

This is a high-level overview of how I use [Cloudflare Workers](https://developers.cloudflare.com/workers/) and [D1](https://developers.cloudflare.com/d1/) to verify and save Lemon Squeezy subscription webhook data. I wanted to do this **without** enabling [Node.js compatibility](https://developers.cloudflare.com/workers/runtime-apis/nodejs/) in the Worker.

<div style="padding: 1rem; border: 1px solid rgba(0, 0, 0, .2); border-radius: 5px; ">

This assumes that you have an understanding of Cloudflare Workers, D1, and have set up a [Lemon Squeezy webhook endpoint](https://docs.lemonsqueezy.com/guides/developer-guide/webhooks#from-the-dashboard) for your product.

</div>

## Webhook Management and Integration

The management of subscription lifecycles in Lemon Squeezy is handled through webhooks, which notify a specified URL upon subscription events—similar to Stripe’s webhook system. For a recent project, I used Cloudflare Workers to accept these webhooks and D1 to manage subscription data in a SQLite database. Opting for a serverless, lightweight architecture was important, especially since this is only handling a few API endpoints.

My requirements for this included:

- A hosted checkout page :white_check_mark:
- Use of a custom domain :white_check_mark:
- Ability to exclusively sell products via in-app purchases, removing them from the main store :white_check_mark:
- Capability to pass user metadata to the checkout page :white_check_mark:
- Follow a similar subscription model to what I'm used to with Stripe :white_check_mark:


## Checkout lifecycle

When users click the upgrade/checkout link within the app, they are redirected to a Lemon Squeezy hosted checkout page. Here, users input their details and payment information to create a subscription.

The URL generated from within the app that points to the hosted checkout page includes necessary user details [passed as query parameters](https://docs.lemonsqueezy.com/help/checkout/passing-custom-data#passing-custom-data-in-checkout-links). For example, the URL might look like this (using template literals in JavaScript/TypeScript):

```ts
`${baseCheckoutUrl}?checkout[email]=${user.email}&checkout[custom][uid]=${user.uid}`
```

These parameters (`checkout[email]` and `checkout[custom][uid]`) pass the user's email and UID to the checkout page, linking them with the newly created subscription. The UID is used as a unique identifier for the user in the database.

{{< warning >}}

There's an edge case to consider: If a user can change their email address or navigates directly to the checkout page without a UID meta param, discrepancies might occur between the webhook payload and the user’s actual details. This issue should be considered when managing webhook responses and communicating with users in post-purchase scenarios.

{{< /warning >}}


## Using Workers Routes

When a subscription is created, Lemon Squeezy sends a webhook to the specified URL. In this setup, the URL is a Cloudflare Worker that receives requests at the `/api/v1/ls/webhook` endpoint.

To consolidate the handling of the site and API under the same domain, I use [Workers Routes](https://developers.cloudflare.com/workers/configuration/routing/routes/) to route `/api/*` to the Cloudflare Worker and `/*` to the main site. This setup allows me to manage all of the site's operations under the same domain, managed by Cloudflare.


{{< image src="images/cf-workers-route.png" class="w-100 mh0 mb3" alt="Cloudflare Workers Routes">}}

<figcaption><small>Cloudflare Workers Routes</small></figcaption>


## Project Structure

This is a high-level overview of the Worker structure. In this app, this is the only API endpoint in `index.ts`, but you can add more routes as needed.

```bash
.
├─ db
│  └─ schema.ts
├─ lib
│  └─ utils.ts
├─ src
│  └─ index.ts
├─ wrangler.toml
...
```




## Using Cloudflare D1

In this example, the database schema is defined in the `schema.ts` file and includes a `customer` table with columns for the `uid`, `email`, `orderId`, `productId`, and `createdAt` timestamp. 


```typescript
// db/schema.ts

import { sql } from 'drizzle-orm';
import { text, integer, sqliteTable } from 'drizzle-orm/sqlite-core';

export const customer = sqliteTable('customer', {
  uid: text('uid').primaryKey(),
  email: text('email').notNull(),
  orderId: text('order_id').notNull(),
  productId: integer('product_id', { mode: 'number' }).notNull(),
  createdAt: text('created_at')
    .notNull()
    .default(sql`CURRENT_TIMESTAMP`),
});
```

The use of D1 is very compelling for this setup since the entire **1/** API is located in Cloudflare, **2/** it's SQL-based and works well with [Drizzle ORM](https://orm.drizzle.team/), and **3/** the overhead to maintain is minimal. The `drizzle-orm` package is used to interact with the database and execute SQL queries.

## Verifying Lemon Squeezy webhooks in a Cloudflare Worker


The `index.ts` file contains the main logic for the Worker. The Worker listens for POST requests on the `/api/v1/ls/webhook` endpoint and verifies the request signature using the secret provided by Lemon Squeezy. 

The Worker uses [Hono](https://hono.dev/), a lightweight framework for Cloudflare Workers, to handle requests. The framework provides middleware, routing, and other features to simplify the development of Workers. The Worker also uses Drizzle to interact with the D1 database. 

```typescript
// src/index.ts

import { Hono } from 'hono';
import { logger } from 'hono/logger';
import { drizzle } from 'drizzle-orm/d1';
import { eq } from 'drizzle-orm';
import { customer } from './../db/schema';

type Env = {
  DB: D1Database;
  LS_WEBHOOK_SECRET: string;
};

const app = new Hono<{ Bindings: Env }>();

app.use(logger());

app.post('/api/v1/ls/webhook', async (c) => {
  if (c.req.method !== 'POST') {
    return c.json({ error: 'Method not allowed' }, 405);
  }

  const secret = c.env.LS_WEBHOOK_SECRET;
  const signature = c.req.header('x-signature');

  if (!signature || !secret) {
    return c.json({ error: 'Unauthorized' }, 401);
  }

  const key = await crypto.subtle.importKey(
    'raw', 
    new TextEncoder().encode(secret), 
    { name: 'HMAC', hash: 'SHA-256' }, 
    false, 
    ['verify']
  );

  const body = await c.req.text();
  const rawBody = new TextEncoder().encode(body);

  const verified = await crypto.subtle.verify(
    'HMAC', 
    key, 
    hexToUint8Array(signature!), 
    rawBody
  );

  if (!verified) {
    return c.json({ error: 'Unauthorized' }, 401);
  }

  const bodyJson = JSON.parse(body);

  const eventName = bodyJson?.meta?.event_name;

  // handle additional events as needed
  if (eventName && eventName !== 'subscription_created') {
    return c.json({ error: 'Event not supported' }, 400);
  }

  // change/adjust as needed
  const guid = bodyJson?.meta?.custom_data?.uid;
  const email = bodyJson?.data?.attributes?.user_email;
  const identifier = bodyJson?.data?.attributes?.identifier;
  const productId = bodyJson?.data?.attributes?.first_order_item?.product_id;

  if (!guid || !email || !identifier || !productId) {
    return c.json({ error: 'Invalid request' }, 400);
  }

  const db = drizzle(c.env.DB);

  await db
    .insert(customer)
    .values({ uid: guid, email, orderId: identifier, productId })
    .onConflictDoUpdate({ target: customer.uid, set: { productId } });
	
  return c.json(200);

});

```

To process the verification, Cloudflare Workers support the [Web Crypto API](https://developers.cloudflare.com/workers/runtime-apis/web-crypto/) through the `SubtleCrypto` interface, which is accessible via `crypto.subtle`. The Worker first checks if the request method is POST and extracts the signature from the request headers. It then verifies the signature using the secret provided by Lemon Squeezy. If the signature is valid, the Worker continues processing the request.

The `hexToUint8Array` function is a utility function that converts a hex string to a `Uint8Array`. `Uint8Array` is used by the [Web Crypto API](https://developer.mozilla.org/en-US/docs/Web/API/Crypto/subtle) for cryptographic operations.

```typescript
// utils.ts
export function hexToUint8Array(hex: string) {
  const matched = hex.match(/.{1,2}/g);
  return new Uint8Array(matched ? matched.map((byte) => parseInt(byte, 16)) : []);
}
```

The `custom_data?.uid` custom data is the user ID passed to the checkout page as a query parameter and is accessed in the [`meta` object of the webhook payload](https://docs.lemonsqueezy.com/help/checkout/passing-custom-data#access-custom-data-in-webhooks).

The Worker then parses the request body as JSON and extracts the event name, user email, order ID, and product ID from the request. The Worker checks if the event name is [`subscription_created`](https://docs.lemonsqueezy.com/guides/developer-guide/webhooks) and inserts the user details into the database using the Drizzle ORM. The Worker returns a `200` status code if the request is successful.

This code also assumes that the D1 database binding is configured in the `wrangler.toml` file. The database configuration includes the database name, database ID, and migrations directory.

```toml
# wrangler.toml
...

[[d1_databases]]
binding = "DB"
database_name = "<database-name>"
database_id = "<database-id>"
migrations_dir = "drizzle"

```

Then, the database configuration can be accessed as an environment variable in the Worker using `c.env.DB` binding.

## Configuring the webhook signing secret

For the Lemon Squeezy webhook request to be verified, the webhook secret signature needs to be stored as a Cloudflare Worker secret and then retrieved from the environment. 

I use the Wrangler CLI to create the secret:

```bash
npx wrangler secret put LS_WEBHOOK_SECRET
```

The secret can then be accessed in the Worker using `c.env.LS_WEBHOOK_SECRET`.

{{< note >}}

The difference between secrets and environment variables is that secrets are encrypted and stored securely by Cloudflare. The environment variables are plaintext and can be accessed by anyone with access to the Worker code (config toml) or console access.

{{< /note >}}

---

This setup uses Cloudflare Workers and D1 to handle Lemon Squeezy subscription webhooks in a serverless architecture. You can easily extend this setup to handle additional events, API routes, and integrate with other services. It's straightforward to set up and maintain, and the lightweight architecture has been ideal so far.

:wave: If you have any questions or feedback, feel free to reach out on [X @siegerts](https://x.com/siegerts).



