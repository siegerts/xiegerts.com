+++
title = "Adding Prisma to Next.js API Routes on Amplify Hosting"
description = "Add Prisma to Next.js API routes deployed to Amplify Hosting."
tags = [
    "development",
    "AWS Amplify",
    "Amplify Hosting",
    "Prisma",
    "PostgreSQL", 
    "API", 
    "Next.js"
]

categories = ["Development", "Serverless"]
date = 2022-06-21T15:25:16-04:00

series = "100 Days of AWS Amplify"
+++

This is a quick follow to [Deploying Next.js SSR and Prisma on Amplify Hosting](/post/prisma-nextjs-amplify-hosting/).

Once you have Prisma deployed and working with Next.js `getServerSideProps`, you can extend the database access into the API routes.

Adjust the default `hello.js` with the below Prisma query. This is the same query as the `index.js` SSR call in the previous post.


```js
// /api/hello.js
import prisma from "@/lib/prisma";

export default async function handler(req, res) {
  const posts = await prisma.post.findMany({
    select: {
      id: true,
      title: true,
      description: true,
    },
  });

  res.status(200).json({ posts });
}
```

The response, if using the _seed_ data will look similar:

```json
// api response
{
  "posts": [
    {
      "id": "cl4ob7ktz00007mupv3jaduyu",
      "title": "First post",
      "description": "Lorem ipsum dolor sit amet"
    },
    {
      "id": "cl4ob7ku100017mupoh1dvigw",
      "title": "Second post",
      "description": "Ipsum dolor sit amet"
    }
  ]
}
```

Add just like that you can now power API routes using Prisma. You can add additional Server Side business logic and call additional external APIs, if needed.

Also, this can be redeployed in the previous app to run on Amplify Hosting :rocket:.

