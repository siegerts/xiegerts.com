+++
title = "Deploying Next.js SSR and Prisma on Amplify Hosting"
description = "Set up and deploy a Next.js SSR app that uses the Prisma ORM with a PostgreSQL database. Once integrated, we'll be able to fetch data using SSR and getServerSideProps. We'll store the connection information for the database in AWS Systems Manager Parameter Store and deploy the app using Amplify Hosting."
tags = [
    "development",
    "AWS Amplify",
    "Amplify Hosting",
    "Prisma",
    "PostgreSQL", 
    "RDS", 
    "Next.js", 
    "React.js",
    "SSM"
]

thumbnail = "images/nextjs-prisma-amplify-hosting.png"
cardType = "summary_large_image"

categories = ["Development", "Serverless"]

date = 2022-06-07T16:03:45-04:00

series = "100 Days of AWS Amplify"

+++

In this post, we'll set up and deploy a [Next.js SSR app](https://nextjs.org/docs/basic-features/data-fetching/get-server-side-props) that uses [Prisma](https://www.prisma.io/) as an ORM with a PostgreSQL database. Once integrated, we'll be able to fetch data using SSR and `getServerSideProps`. We'll store the connection information for the database in [AWS Systems Manager Parameter Store](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html) and deploy the app on [Amplify Hosting](https://aws.amazon.com/amplify/hosting/).

#### The application and deployment
1. Create a Next.js app 
2. Set up Prisma and seed the DB
3. Create and store the production DB connection string in Parameter Store (SSM)
4. Configure the Amplify Hosting build to retrieve the connection string from (SSM)
5. Deploy!


You'll need a few prerequisites to follow along:

- A GitHub account to deploy the app from (with Amplify CI/CD)
- A database supported by Prisma (PostgreSQL)
- An AWS account and some familiarity with AWS Systems Manager Parameter Store


The code associated with this post is available https://github.com/siegerts/amplify-hosting-ssr-prisma.

To follow along, you should be familiar with Next.js, SSR, and relational databases. Let's get up and running on Amplify Hosting ⚡!


## Create a new Next.js project 

We'll create a the default Next.js SSR application using `create-next-app`. You can also use the steps below (after the set up) if you're adding Prisma to a an existing Next.js application deployed on Amplify Hosting.

```
npx create-next-app amplify-prisma
```

For reference, below is the `package.json`. The structure of the project should be configured for SSR. The `scripts` section should match below:

```json
 {
  "name": "amplify-prisma",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint"
  },
  "dependencies": {
    "next": "12.1.6",
    "react": "18.1.0",
    "react-dom": "18.1.0"
  },
  "devDependencies": {
    "eslint": "8.17.0",
    "eslint-config-next": "12.1.6"
  }
}
```

It's important to note that the app will be built as an SSR application and not a static (SSG) application. 

{{< note >}}

It's good to familiarize yourself with the [Amplify Hosting Next.js SSR support documentation](https://docs.aws.amazon.com/amplify/latest/userguide/server-side-rendering-amplify.html#ssr-Amplify-support). The docs cover the supported features, functionality, and how to configure Next.js apps for SSG or SSR on Amplify Hosting.

Another good resource is the [Amplify Hosting FAQ](https://github.com/aws-amplify/amplify-hosting/blob/main/FAQ.md#ssr) on GitHub. 

{{< /note >}}


Prisma requires the application to have some server-side (i.e. backend) layer to run in. Also, Amplify Hosting will deploy the app differently if depending on the type (SSR vs SSG). If starting a new application, the directory structure will resemble:

```diff
  .
  ├── node_modules/
  ├── pages/
  │   ├─ api/
  │   ├─ _app.js
  │   └─ index.js
  ├── public/
  ├── styles/
  ├── next.config.js 
  ...

  └─ README.md
```

### Setting up Prisma 

Prisma will allow us to connect to our PostgreSQL database. We'll follow along with the [Prisma documentation](https://www.prisma.io/docs/getting-started/setup-prisma/start-from-scratch/relational-databases/connect-your-database-node-postgres) to get the database set up.

1. Add Prisma as a development dependency (i.e. `devDependencies`)
```
yarn add prisma -D
```

2. Add the client as a dependency
```
yarn add @prisma/client
```


3. Verify the installation
```
npx prisma
```

4. Initialize Prisma within the project
```
npx prisma init
```

From the Prisma documentation, this does two things:

> 1. Creates a new directory called prisma that contains a file called schema.prisma, which contains the Prisma schema with your database connection variable and schema models
> 2. Creates the .env file in the root directory of the project, which is used for defining environment variables (such as your database connection)


After running `npx prisma init`, the new files will be added.

```diff
  .
  ├── node_modules/
+ ├── prisma/
+ │   └─ prisma.schema
  ├── pages/
  │   ├─ api/
  │   ├─ _app.js
  │   └─ index.js
  ├── public/
  ├── styles/
  ├── next.config.js 
+ ├── .env
  ...

  └─ README.md
```

{{< warning >}}

**Add** `.env` to the `.gitignore` file! You don't want to check this in to source control!

{{< /warning >}}

<br />

#### prisma.schema

The schema file is used to define the structure of the database through tables and relationships. Prisma then uses this structure to interact with the database with a generated client (below) that provides a typed-ORM specific to the naming conventions and table attributes that you define.

For now, we'll create a basic schema with a `Post` table that has a few attributes. In `prisma.schema`, add the following:

```graphql
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model Post {
  id          String   @id @default(cuid())
  title       String?  @db.Text
  description String?  @db.Text
  content     String?  @db.Text
  slug        String   @default(cuid())
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
  hidden      Boolean  @default(false)
}
```

#### Setting the local `DATABASE_URL` environment variable

You'll need to update the `DATABASE_URL` environment variable to point to a PostgreSQL compatible database. This database can be different from what you'll use when you deploy the application. Typically, you'll use a different database for each environment (local, test, prod, etc.). 

No matter what you use, **make sure not to expose this!** For example, you may use something like this in the `.env` file:

```ini
DATABASE_URL="postgresql://username:somepassword@cloudserver:5432/mydb?schema=public"
```


#### Initialize the database schema

Now, generate the Prisma client and push the schema in the application to the database. The client generation needs to happen each time the schema changes.

```
npx prisma generate
```

Create the database tables based on the schema file. 

```
npx prisma db push
```

If successful, the database will now contain the tables representing the local schema file. A quick way to check this is to use the Prisma Studio UI. Run `npx prisma studio` to open browser-based database viewer on `http://localhost:5555/`.


### Instantiating the Prisma client

Now, to make Prisma available in the the Next.js app, add a `lib` directory with `prisma.js`.

```diff
  .
  ├── node_modules/
+ ├── lib/
+ │   └─ prisma.js
  ├── prisma/
  │   └─ prisma.schema
  ├── pages/
  │   ├─ api/
  │   ├─ _app.js
  │   └─ index.js
  ├── public/
  ├── styles/
  ├── next.config.js 
  ├── .env
  ...

  └─ README.md
```

Add the following in `prisma.js`:

```js 
// lib/prisma.js
import { PrismaClient } from "@prisma/client";

let prisma;

if (process.env.NODE_ENV === "production") {
  prisma = new PrismaClient();
} else {
  if (!global.prisma) {
    global.prisma = new PrismaClient();
  }
  prisma = global.prisma;
}

export default prisma;
```

Now, we can use this in `getServerSideProps` within pages and API routes to fetch the data from the database.

But first, we'll add some test data to the database to help verify that the data fetching works. You can enter data manually into the database or use the Prisma `seed` utility. To use the utility, add a `seed.js` file to the `prisma` directory:

```diff
  .
  ├── node_modules/
  ├── lib/
  │   └─ prisma.js
  ├── prisma/
+ │   ├─ seed.js 
  │   └─ prisma.schema
  ├── pages/
  │   ├─ api/
  │   ├─ _app.js
  │   └─ index.js
  ├── public/
  ├── styles/
  ├── next.config.js 
  ├── .env
  ...

  └─ README.md
```

Then, in `seed.js`, add a few records to the database using the Prisma client.

```js 
// seed.js 
const { PrismaClient } = require("@prisma/client");

const prisma = new PrismaClient();

async function main() {
  let posts = [
    {
      title: "First post",
      description: "Lorem ipsum dolor sit amet",
      content:
        "Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.",
      slug: "first-post-1",
    },
    {
      title: "Second post",
      description: "Ipsum dolor sit amet",
      content:
        "Tristique sollicitudin nibh sit amet commodo. Feugiat vivamus at augue eget",
      slug: "second-post-2",
    },
  ];

  await prisma.post.createMany({
    data: posts,
  });
}

main()
  .catch((e) => {
    console.error(e);
    process.exit(1);
  })
  .finally(async () => {
    await prisma.$disconnect();
  });

```

Additionally, add the `seed` command into `prisma` section in `package.json`:

```js
"prisma": {
  "seed": "node prisma/seed.js"
},
```

Now, use the below command to seed the data:

```
npx prisma db seed
```

Cool! Now we have test data. Once again, a quick way to check this is to use the Prisma Studio UI. Run `npx prisma studio` to open browser-based database viewer on `http://localhost:5555/`.

## Testing the Prisma setup locally

To test the data fetching locally, we'll add a simple database query using Prisma to the `index.js` page of the app.

Replace the contents of `index.js` with the snippet below:

```jsx
// index.js
import Head from "next/head";
import styles from "../styles/Home.module.css";
import prisma from "../lib/prisma";

export const getServerSideProps = async () => {
  // this query grabs the data!
  const posts = await prisma.post.findMany({
    select: {
      id: true,
      title: true,
      description: true,
    },
  });
  return { props: { posts } };
};

export default function Home({ posts }) {
  return (
    <div className={styles.container}>
      <Head>
        <title>Create Next App</title>
        <meta name="description" content="Amplify Hosting + Prisma!" />
        <link rel="icon" href="/favicon.ico" />
      </Head>

      <main className={styles.main}>
        <h1 className={styles.title}>Amplify + Prisma!</h1>
        <div className={styles.grid}>
          {posts &&
            posts.map((post) => (
              <div key={post.id} className={styles.card}>
                <h2>{post.title}</h2>
                <p>{post.description}</p>
              </div>
            ))}
        </div>
      </main>
    </div>
  );
}
```

This query will select all the `posts` from the `Post` table and only return the columns specified in `select`. Test the app with `yarn dev`. Great - The app should be working! 

Next, deploy to Amplify Hosting 🚀.


## Deploy the app on Amplify Hosting

A few things need to happen before we deploy the app. We're going to need to push the application to GitHub since we'll deploy with the continuous CI/CD in Amplify Hosting.

Also, we're going to retrieve the production database (or _non-development_) connection string from AWS Parameter Store. You could also follow a similar approach for use with Secrets Manager if you'd rather use that service. From the Parameter Store documentation:

> To implement password rotation lifecycles, use AWS Secrets Manager. You can rotate, manage, and retrieve database credentials, API keys, and other secrets throughout their lifecycle using Secrets Manager.

The pattern that we'll use can work with either service. I prefer Parameter Store _unless_ I need rotation as mentioned in the documentation snippet.


#### Create the database connection string parameter


Before deploying the app, create a `SecureString` parameter in Parameter Store. Typically, I use a Python helper ([Appendix](#appendix-add-securestring-in-parameter-store-with-python-and-boto3)) script to quickly create parameters but you can also manually add in the AWS Console.

The parameter should follow the connection string pattern that we've been using and is required by Prisma:

```ini
DATABASE_URL="postgresql://username:somepassword@cloudserver:5432/mydb?schema=public"
```

Make sure that the **service role** that you use or create during the deployment process (below) has the correct permissions to access this parameter in SSM. 

If you need to adjust the role with an inline policy for a specific resource, you can use the below:

```json
{
    "Sid": "AmplifySSMCalls",
    "Effect": "Allow",
    "Action": [
        "ssm:GetParametersByPath",
        "ssm:GetParameters",
        "ssm:GetParameter",
    ],
    "Resource": "arn:aws:ssm:*:*:path/to/parameter*"
},
```

This is something to keep in mind for Next.js SSR applications on Amplify - you want to make sure that they Backend Role has the correct permissions for your application.

The permissions required for Next.js apps on Amplify are listed in [SSR IAM Permissions]( https://docs.aws.amazon.com/amplify/latest/userguide/server-side-rendering-amplify.html#ssr-IAM-permissions).


#### Update-env.sh

We'll use this in the Amplify build settings to:

1. Retrieve the database connection string for Prisma
2. Inject that connection string into the `.env` file as an environment variable to have it picked up by `npx prisma db generate` 

That way, the Prisma client is generated using the correct production database. Add the `update-env.sh` file into the root directory:

```diff
  .
  ├── node_modules/
  ├── lib/
  ├── prisma/
  ├── pages/
  ├── public/
  ├── styles/
  ├── next.config.js 
  ├── .env
+ ├── update-env.sh
  ...

  └─ README.md
```

In the file, add the below command to call the AWS SDK (`aws ssm ...`) to retrieve the saved parameter. Make sure that the parameter name (i.e. `'/path/to/parameter'`) matches the name that you used when creating the parameter.

```sh
#!/bin/bash

echo "DATABASE_URL=$(aws ssm get-parameter --name '/path/to/parameter' --with-decryption | jq '.Parameter.Value')" >> .env
```

The script will retrieve the parameter that you've created in Parameter Store and write it in the `.env` file. 

{{< note >}}

This command is executed within a script to _avoid having the output value printed to standard out_ in the build logs.

{{< /note >}}

Now, lets create the app.


### Create a new app in Amplify 

We'll use the _Host a web app_ flow in Amplify Hosting. When creating the application there are a few things to configure.

1. Connect to the application repository in your GitHub account

2. Make sure the app is recognized as a **Server-side rendering deployment** 

3. Choose a **role** that has the correct permissions for SSM (see section above)

4. **Update the build and test settings** to run the `update-env.sh` script (below)

5. Click on **Advanced Settings** and update the application _environment variables_

<br />

#### Update the build and test settings

You'll need to modify the Amplify Build settings in your app to include the the utility script that was just created above. The Prisma client is generated using the secrets retrieved with `update-env.sh`.

```diff
version: 1
frontend:
  phases:
    preBuild:
      commands:
+       - yum -y install jq # 1
+       - jq --version # 2
+       - nvm install 16 # 3
        - yarn install
    build:
      commands:
+       - bash update-env.sh # 4
+       - npx prisma generate # 5
        - yarn run build
  artifacts:
    baseDirectory: .next
    files:
      - '**/*'
  cache:
    paths:
      - node_modules/**/*
```

When the the build runs:

1. [`jq`](https://stedolan.github.io/jq/) will be installed as a utility to help parse the AWS CLI response from SSM

2. Verify `jq` is installed by printing version

3. Switch with Node.js 16

4. Execute the `update-env.sh` utility to retrieve the database connection string from SSM and write the value to the `.env` file

5. Generate the `prisma client` using the connection string SSM parameter


#### Update environment variables

Set the [environment variable `AMPLIFY_NEXTJS_EXPERIMENTAL_TRACE=true`](https://github.com/aws-amplify/amplify-hosting/blob/main/FAQ.md#webpack-modulenotfound-errors) in your App settings.  In the Environment variables section for the new application, set:

- Key = `AMPLIFY_NEXTJS_EXPERIMENTAL_TRACE` 
- Value = `true`


## Application deployment

Finish the configuration set up and **Create** the application. In the build logs, you'll see references to the modified build settings from above:

```
2022-06-020T15:23:48.487Z [INFO]: # Executing command: bash update-env.sh
2022-06-0215:24:01.809Z [INFO]: # Executing command: npx prisma generate
2022-06-02T15:24:03.120Z [INFO]: Environment variables loaded from .env
2022-06-02T15:24:03.122Z [INFO]: Prisma schema loaded from prisma/schema.prisma
```


## Conclusion

Great! The application is now set up to query data from a relational PostgreSQL database using Prisma. You should now see the data populate the index page when you load deployment URL generated by Amplify Hosting. Additionally, the database connection details are securely stored in AWS Systems Manager Parameter Store 🔒.



<br />
<br />


## Appendix: Add `SecureString` in Parameter Store with Python and `boto3`

The utility functions below create `SecureString` parameters using Python.

```python
import boto3
import logging
from botocore.exceptions import ClientError

session = boto3.Session()

def put_parameter(parameter_name, parameter_value, parameter_type):
    ssm_client = session.client('ssm')
    try:
        result = ssm_client.put_parameter(
            Name=parameter_name,
            Value=parameter_value,
            Type=parameter_type
        )
    except ClientError as e:
        logging.error(e)
        return None
    return result['Version']


def get_parameter(parameter_name, with_decryption):
    ssm_client = session.client('ssm')
    try:
        result = ssm_client.get_parameter(
            Name=parameter_name,
            WithDecryption=with_decryption
        )
    except ClientError as e:
        logging.error(e)
        return None
    return result
   
    
    
if __name__ == "__main__":
    version = put_parameter("</path/to/parameter>", value, "SecureString")
    print(version)

```