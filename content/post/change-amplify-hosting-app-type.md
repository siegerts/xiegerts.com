+++
title = "Changing a Next.js SSG App to SSR on Amplify Hosting"
description = "Change the application type in Amplify Hosting using the AWS CLI"
tags = [
    "development", 
    "AWS CLI", 
    "Amplify Hosting", 
    "SSR", 
    "AWS Amplify", 
    "SSG"
]

categories = ["Development"]

series = "100 Days of AWS Amplify"


date = 2022-09-26T10:03:45-04:00
+++


If you want to change to the application type from SSR to SSG, or SSG to SSR, then the `plaform` and `framework` type needs to be changed in your application settings in Amplify Hosting. If not, you'll likely hit deploy errors or 503s  when the site builds and deploys. 

The current framework type is listed in the **App settings** -> **Building settings** -> **app build specification**. This is determined when the application is created. For Next.js SSR apps, the type is selected if the build command in `package.json` is `next build`. The match criteria for each app build type is explained in the [Deploying a Next.js SSR app with Amplify](https://docs.aws.amazon.com/amplify/latest/userguide/server-side-rendering-amplify.html#redeploy-ssg-to-ssr) documentation.

 
### Change an SSG app to SSR

Use the **AWS CLI** to modify the framework value for the application.

{{< warning >}}
Use the AWS CLI and not the Amplify CLI for this step!
{{< /warning >}}

#### App Type 

First update the app to have a `--platform` type of `WEB_DYNAMIC`.

Then, change the `--framework` to `Next.js - SSR` if the app is not using an Amplify Backend.

```bash
aws amplify update-app --app-id <APP_ID> --platform WEB_DYNAMIC --region <REGION>

aws amplify update-branch --app-id <APP_ID> --branch-name <BRANCH_NAME> --framework 'Next.js - SSR' --region <REGION>
```

#### SSR with Amplify Backend

Change the `--framework` to `Next.js - SSR - Amplify` if the application is using an Amplify Backend.

#### App Build settings

Next, adjust the application build settings to reflect an SSR application. As noted above, Amplify sets the app type when the app is first created, so this will need updated for SSR.

As an example, it should resemble:

```yaml
version: 1
frontend:
  phases:
    preBuild:
      commands:
        - yarn install
    build:
      commands:
        - yarn run build
  artifacts:
    baseDirectory: .next
    files:
      - '**/*'
  cache:
    paths:
      - node_modules/**/*
```

### Change an SSR app to SSG

If you need to change the app to SSG, then follow the steps outlined in the [Amplify Hosting FAQ - Convert an SSR App to SSG](https://github.com/aws-amplify/amplify-hosting/blob/main/FAQ.md#convert-an-ssr-app-to-ssg).


### Rewrites

The rewrite will be added after the SSR app is built/deployed. You should already have one of the rewrite lines in that config if one of your branches is SSR and successfully deploying.


