+++
title = "Building and deploying a Slack app with Python, Bolt, and AWS Amplify"
description = "Building and deploying a Slack app with AWS Amplify, Python, and Bolt."

tags = [
    "development", 
    "AWS Amplify", 
    "Slack",
    "Slackbot",
    "Lambda",
    "full stack",
    "serverless",
    "python",
    "slash command",
    "lambda",
    "bolt-python",
    "deployment"
]

thumbnail = "images/slack-python-amplify-header.png"
cardType = "summary_large_image"
date = 2021-05-24T21:47:41-04:00
categories = ["Development", "Serverless"]
series = "100 Days of AWS Amplify"


+++

{{< note >}}

This post is part of an ongoing [series](/series/100-days-of-aws-amplify/) focused on tips and tricks when building fullstack serverless apps with [AWS Amplify](https://docs.amplify.aws/).

{{< /note >}}


Slack apps and bots are a handy way to automate and streamline repetitive tasks. Slack's [Bolt](https://api.slack.com/start/building/bolt-python) framework consolidates the different mechanisms to capture and interact with Slack events into a single library. 

From the [Slack Bolt documentation](https://api.slack.com/tools/bolt):

> _All flavors of Bolt are equipped to help you build apps and integrations with our most commonly used features._

> _Install your app using OAuth 2.0_
> _Receive real time events and messages with the Events API_
> _Compose rich, interactive messages with Block Kit_
> _Respond to slash commands, shortcuts, and interactive messages_
> _Use a wide library of Web API methods_

<br />

In my experience, this has made it straight forward to combine functionality instead of having multiple apps all handling different elements of Slack's API ecosystem.

There is a **ton** that you can do combining this library with the Amplify toolchain. So, let's create an app using Python :snake:!

We'll create a simple Slack app that's triggered when the `/start-process` slash command is run from within Slack.

{{< image src="images/slack-command-type.png" class="w-80 mh0 mb3 ba b--near-white" >}}

<br />


# Set up your Amplify project

This is the standard procedure. The project name used is `slackamplify`.

```bash
? Enter a name for the project slackamplify
The following configuration will be applied:

Project information
| Name: slackamplify
| Environment: dev
| Default editor: Visual Studio Code
| App type: javascript
| Javascript framework: none
| Source Directory Path: src
| Distribution Directory Path: dist
| Build Command: npm run-script build
| Start Command: npm run-script start

```
<br />

## Add a REST API with a Lambda function

The [REST API](https://docs.amplify.aws/cli/restapi) is the lifeline of the Slack app. This API will be called each time the slash command is invoked from within Slack. 

```bash
amplify add api
```

For the API path, I use a base path (`/`) below. If you something specific here, make sure to append that path to your base URL endpoint when configuring the Slack app below.

```bash
? Please select from one of the below mentioned services: REST
? Provide a friendly name for your resource to be used as a label for this category in the project: slac
kpython
? Provide a path (e.g., /book/{isbn}): /
? Choose a Lambda source Create a new Lambda function
? Provide an AWS Lambda function name: slackpythonfunction
? Choose the runtime that you want to use: Python
Only one template found - using Hello World by default.

Available advanced settings:
- Resource access permissions
- Scheduled recurring invocation
- Lambda layers configuration

? Do you want to configure advanced settings? No
? Do you want to edit the local lambda function now? No
Successfully added resource slackpythonfunction locally.
```

<br />


## Set up the function's virtual environment 

On to the function!

But first, we need to update the function's Python virtual environment.


### Install the Python dependencies

The next step is **very important**!

Change into the function's directory!

```
cd <project>/amplify/backend/function/<function-name>
```

To install dependencies, you'll need to first activate the Python virtual environment (below) for this function. To do this, you need to be in the function directory (step above). I've seen this trip folks up when folks actually install dependencies into the global Python installation and not into the correct virtual environment. The Amplify CLI will will _prep_ your function directory for [Pipenv](https://pipenv.pypa.io/en/latest/) if you select the Python runtime for your function during the `amplify add api` prompts.

{{< tip >}}

A quick reminder - Pipenv differs slightly from other Python virtual environment tooling:

- `pipenv shell` activates the environment
- `exit` deactivates the environment

{{< /tip >}}


### Activate the virtual environment

```bash
pipenv shell
```

I like to think of each Amplify function as it's own isolated environment where the code (i.e. functionality) pairs `1:1` with the dependencies within that environment. And for each function, you can have your own isolated Python virtual environment environment.


{{< image src="images/virtual-environment.png" class="w-100 mh0 mb3" >}}


{{< note >}}

If you want to share dependencies, or code, across functions then you'll want to check out the [Lambda Layers](https://docs.amplify.aws/cli/function/layers) capability in the Amplify CLI to see if that fits your use case.

{{< /note >}}


Now, install the required dependencies:

<div class="mv4">

```bash
pipenv install slack_bolt
```

</div>


After installing the first dependency, you will see a **Pipfile.lock** file in the directory.  

<div class="mv4">

```bash
pipenv install python-lambda
```

</div>

The **Pipfile** will now list the above dependencies that you installed.

```diff
  [[source]]
  name = "pypi"
  url = "https://pypi.org/simple"
  verify_ssl = true

  [dev-packages]

  [packages]
  src = {editable = true, path = "./src"}
+ slack-bolt = "*"
+ python-lambda = "*"

  [requires]
  python_version = "3.8"

```

<br />

## Update the Lambda function

At this point, we'll use the the placeholder Lambda function from the [Example with AWS Lambda](https://slack.dev/bolt-python/concepts#lazy-listeners) in the **Bolt** documentation. It may be collapsed _and at the bottom of the page_, so double check that you're looking at the right snippet.


```python
# reference example from: https://slack.dev/bolt-python/concepts#lazy-listeners

from slack_bolt import App
from slack_bolt.adapter.aws_lambda import SlackRequestHandler

# process_before_response must be True when running on FaaS
app = App(process_before_response=True)

def respond_to_slack_within_3_seconds(body, ack):
    text = body.get("text")
    if text is None or len(text) == 0:
        ack("Usage: /start-process (description here)")
    else:
        ack(f"Accepted! (task: {body['text']})")

import time
def run_long_process(respond, body):
    time.sleep(5)  # longer than 3 seconds
    respond(f"Completed! (task: {body['text']})")

app.command("/start-process")(
    ack=respond_to_slack_within_3_seconds,  # responsible for calling `ack()`
    lazy=[run_long_process]  # unable to call `ack()` / can have multiple functions
)

def handler(event, context):
    slack_handler = SlackRequestHandler(app=app)
    return slack_handler.handle(event, context)

```

I recommend reading through [this GitHub](https://github.com/slackapi/bolt-python/issues/335#issuecomment-837095097) thread to get a better understanding of `process_before_response`, `ack()`, and `lazy=` in the code above. These elements make it possible to run this an application like this in a serverless environment without the need for a persistent long-running server.

<br />

### Adjusting the function's IAM policy 

You'll need to add the adjust policy below for your Lambda function as mentioned in the **Bolt** docs.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "lambda:InvokeFunction",
                "lambda:GetFunction"
            ],
            "Resource": "*"
        }
    ]
}
```

<div class="mv4">

To do so, adjust the `lambdaexecutionpolicy` in the **<project>/amplify/backend/function/\<function-name\>/\<function-name\>-cloudformation-template.json** to add the above statement.

</div>

```diff
"lambdaexecutionpolicy": {
    "DependsOn": ["LambdaExecutionRole"],
    "Type": "AWS::IAM::Policy",
    "Properties": {
        "PolicyName": "lambda-execution-policy",
        "Roles": [{ "Ref": "LambdaExecutionRole" }],
        "PolicyDocument": {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Effect": "Allow",
                    "Action": ["logs:CreateLogGroup",
                    "logs:CreateLogStream",
                    "logs:PutLogEvents"],
                    "Resource": { "Fn::Sub": [ "arn:aws:logs:${region}:${account}:log-group:/aws/lambda/${lambda}:log-stream:*", { "region": {"Ref": "AWS::Region"}, "account": {"Ref": "AWS::AccountId"}, "lambda": {"Ref": "LambdaFunction"}} ]}
                 },
+                {
+                   "Sid": "VisualEditor0",
+                   "Effect": "Allow",
+                   "Action": [
+                      "lambda:InvokeFunction",
+                      "lambda:GetFunction"
+                   ],
+                   "Resource": "*"
+               }
             ]
         }
     }
 }

```

<br />

## Push the API 

```
amplify push
``` 

And create the resources :rocket:

```                                 
✔ Successfully pulled backend environment dev from the cloud.

Current Environment: dev

| Category | Resource name       | Operation | Provider plugin   |
| -------- | ------------------- | --------- | ----------------- |
| Function | slackpythonfunction | Create    | awscloudformation |
| Api      | slackpython         | Create    | awscloudformation |
? Are you sure you want to continue? (Y/n) Yes
```

<div class="mv4">
Once this is deployed, the URL can be used in the next step (Create the Slack app).

</div>

```bash
REST API endpoint: https://<id>.execute-api.<region>.amazonaws.com/dev
```



We now have 90% of the Slack app deployed. The app needs environment variables to be complete but we can only get those once we create a new app in Slack. So, let's do that. 

<br />

## Create the Slack app

Now, you'll need to head over to Slack to create (if it doesn't exist) and link the app to the Amplify API.

1. Go to https://api.slack.com/apps and **Create new app** > **From scratch**. Use the endpoint from the API above!



{{< image src="images/create-new-command.png" class="w-80 mh0 mb3 ba b--near-white" >}}

<br />

2. **Select Name & workspace**

{{< image src="images/name-workspace.png" class="w-80 mh0 mb3" >}}

<br />

3. **Basic information** > **Add features & functionality** > **Slash commands**

{{< image src="images/name-workspace.png" class="w-80 mh0 mb3" >}}

<br />

4. **Basic information** > **Install your app**

{{< image src="images/install-slack-app.png" class="w-80 mh0 mb3 ba b--near-white" >}}

<br />

## Update the Lambda function's environment variables

Slack will generate the required tokens and secrets that you'll need to populate into your function with environment variables once the app and command are set up.


Add the below variable/token pairs to the function in Lambda. To add environment variables, go to **Configuration** > **Environment variables** in the AWS Console for the Lambda function.


- `SLACK_SIGNING_SECRET`: **Basic information** > **App credentials**
- `SLACK_BOT_TOKEN`     : **OAuth & Permissions** > starts with `xoxo-`

<br />

{{< image src="images/environment-variables-lambda.png" class="w-80 mh0 mb3 ba b--near-white" >}}

<br />


## Test the command in Slack

Awesome! Now let's test the slash command. 


It shows in the command menu.

{{< image src="images/slack-command-type.png" class="w-80 mh0 mb3 ba b--near-white" >}}

<br />

And...the command runs as expected when submitted along with some data.

<br />

{{< image src="images/execute-command.png" class="w-80 mh0 mb3 ba b--near-white" >}}

<br />

## Conclusion

There you have it! A Slack app running in Lambda created with AWS Amplify running in a serverless context.

I'd definitely take a look at the other ways that you can [listen and respond](https://github.com/SlackAPI/bolt-python) to messages. My favorite? Responding to reactions...

<div class="mv4">

```python
@app.message(":wave:")
def say_hello(message, say):
    user = message['user']
    say(f"Hi there, <@{user}>!")
```

</div>

{{< tip >}}

Once you start listening for other events, you'll need to verify your endpoint configuration. For example, some events require that the endpoint URL end with `/slack/events`.

{{< /tip >}}


Hope that helps! :rocket:







