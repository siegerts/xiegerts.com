+++
title = "NextAuth.js with Slack Provider Set Up"
description = "Set up NextAuth.js with the Slack provider and Prisma."
tags = ["Next.js", "Sign in with Slack", "Prisma.js"]

categories = ["Development"]
date = 2021-12-18T19:33:55-05:00
+++

Just a quick note on configuration changes needed for the [Slack provider](https://next-auth.js.org/providers/slack) to save the authenticated user correctly using the Prisma adapter.

The below changes assume that the app is set up following the NextAuth.js documentation.

### `schema.prisma`

The schema needs to be adjusted to include both `state` and `ok`.

```diff

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model Account {
  id                 String   @id @default(cuid())
  userId             String
  type               String
  provider           String
  providerAccountId  String
  refresh_token      String?
  access_token       String?
  expires_at         Int?
  token_type         String?
  scope              String?
  id_token           String?
  session_state      String?
+ ok                 Boolean?
+ state              String?
  oauth_token_secret String?
  oauth_token        String?

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([provider, providerAccountId])
}

model Session {
  id           String   @id @default(cuid())
  sessionToken String   @unique
  userId       String
  expires      DateTime
  user         User     @relation(fields: [userId], references: [id], onDelete: Cascade)
}

model User {
  id            String    @id @default(cuid())
  name          String?
  email         String?   @unique
  emailVerified DateTime?
  image         String?
  accounts      Account[]
  sessions      Session[]
}

model VerificationToken {
  identifier String
  token      String   @unique
  expires    DateTime

  @@unique([identifier, token])
}
```

### `api/auth/[...nextauth].js`

```JavaScript

import NextAuth from "next-auth";
import SlackProvider from "next-auth/providers/slack";
import { PrismaAdapter } from "@next-auth/prisma-adapter";
import prisma from "../../../lib/prisma";

export default NextAuth({
  adapter: PrismaAdapter(prisma),
  providers: [
    SlackProvider({
      clientId: process.env.SLACK_CLIENT_ID,
      clientSecret: process.env.SLACK_CLIENT_SECRET,
      idToken: true,
    }),
  ],
  callbacks: {
    async session({ session, token, user }) {
      // ...
      return session;
    },
  },
  secret: process.env.SECRET,
  theme: {
    colorScheme: "light",
  },
  theme: {
    colorScheme: "auto",
    logo: "https://next-auth.js.org/img/logo/logo-sm.png",
    brandColor: "#1786fb",
  },
});



```

If you have any questions or feedback, feel free to reach out on [X @siegerts](https://x.com/siegerts).