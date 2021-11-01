+++
title = "Deploying a FastAPI backend using AWS Amplify Container-based REST APIs"
description = "Deploying a FastAPI backend using AWS Amplify Container-based REST APIs."

tags = [
    "Docker", 
    "AWS Amplify", 
    "FastAPI",
    "Fargate",
    "full stack",
    "serverless",
    "API",
    "python",
    "data science",
    "analytics",
    "auto scale",
]

thumbnail = "images/fargate-fastapi-header.png"
cardType = "summary_large_image"

date = 2021-05-11T23:34:21-04:00
categories = ["Development", "Serverless"]
series = "100 Days of AWS Amplify"


#[[resources]]
#  name = "amplify-environment"
#  src = "../images/amplify-env-terminal.png"
#  title = "Amplify environment terminal"

+++

{{< note >}}

This post is part of an ongoing [series](/series/100-days-of-aws-amplify/) focused on tips and tricks when building fullstack serverless apps with [AWS Amplify](https://docs.amplify.aws/).

{{< /note >}}


No surprise that it's a difficult process to go from development to production deployments in an organization :shrug:. 

This is especially true for analytics and data science teams developing models or business logic. There's usually not a clear path or tooling to support application hand-offs or deployments across teams.

One way to bridge the gap between applications is to allow for service-to-service communication. REST APIs are great for this type of use-case. Especially when auto-generated, interactive documentation, is available out-of-the-box :wink:.

**AWS Amplify provides a great option for this - Serverless containers using API Gateway + AWS Fargate.**


This guide will follow the steps outlined in the [Serverless containers](https://docs.amplify.aws/cli/usage/containers) section of the Amplify documentation and the [FastAPI Docker Deployment](https://fastapi.tiangolo.com/deployment/docker/) documentation to quickly create and deploy a production-ready, scalable REST API.


### Prerequisites
You'll need to have the below installed and configured.

- [AWS Amplify CLI](https://docs.amplify.aws/cli/start/install#install-the-amplify-cli) - working with Amplify (I'm using `v4.50.2` )
- [Docker](https://www.docker.com/products/docker-desktop) - testing the APIs locally



### FastAPI

As mentioned above, we'll use FastAPI as the backend framework. The application pattern will feel natural if you're familiar with [Flask](https://flask.palletsprojects.com/en/2.0.x/).

FastAPI has quickly become a go-to framework for setting up APIs for data science and analytics-based workloads. From the project page: 

> _FastAPI is a modern, fast (high-performance), web framework for building APIs with Python 3.6+ based on standard Python type hints._

Among some of the callouts:

- _Fast: Very high performance, on par with NodeJS and Go_ 
- _Robust: Get production-ready code. With automatic interactive documentation._


This is a great option for data science teams that want to focus on productionalizing APIs with the ability to scale in the future if needed.

Also, as we'll see below, testing locally is a smooth process using Docker.


### Serverless containers with Amplify

If you need an escape hatch from just using Lambda then this may be a great option for you. We'll walk through a more basic set up below but the same approach can be leveraged for more complex backends using a **docker-compose.yml** configuration.

> Serverless containers provide the ability for you to deploy APIs and host websites using AWS Fargate. Customers with existing applications or those who require a lower level of control can bring Docker containers and deploy them into an Amplify project fully integrating with other resources.



## Project set up 

Okay, let's deploy an API.

First, create a new directory and initialize the the Amplify project. In this example, I'm not hooking up a frontend so I just select the default options. **But**, if a frontend is something that you're interested in, then it's possible to layer that on and have the UI (React, Vue, etc.) talk to the backend API that is being created.

Make the directory.
 ```bash
 mkdir fastapi-amplify
 ```

Initialize the Amplify project.
```bash
amplify init
```

<details>
  <summary>Terminal workflow</summary>

```
Scanning for plugins...
Plugin scan successful
Note: It is recommended to run this command from the root of your app directory
? Enter a name for the project fastapiamplify
? Enter a name for the environment dev
? Choose your default editor: Visual Studio Code
? Choose the type of app that you're building javascript
Please tell us about your project
? What javascript framework are you using none
? Source Directory Path:  src
? Distribution Directory Path: dist
? Build Command:  npm run-script build
? Start Command: npm run-script start
```

</details>


## Enabling container-based deployments

Now, with the project initialized, we need to enable the use of the container-based deployment options.

```bash
amplify configure project
```

{{< image src="images/container-based-deployment.png" class="w-100 mh0 mb3" >}}


<details>
  <summary>Terminal workflow</summary>

  ```
  ? Which setting do you want to configure? Advanced: Container-based deployments
  Using default provider  awscloudformation
  ? Do you want to enable container-based deployments? Yes

  ```

</details>

## Adding the API 

With the project configured, let's add the soon-to-be API.

```
amplify add api
```

{{< image src="images/add-api.png" class="w-100 mh0 mb3" >}}


<details>
  <summary>Terminal workflow</summary>

```
? Please select from one of the below mentioned services: REST
? Which service would you like to use API Gateway + AWS Fargate (Container-based)
? Provide a friendly name for your resource to be used as a label for this category in the project: fastapirest
? What image would you like to use Custom (bring your own Dockerfile or docker-compose.yml)
? When do you want to build & deploy the Fargate task On every "amplify push" (Fully managed container source)
? Do you want to restrict API access No
Successfully added resource fastapirest locally.
```

</details>

The CLI will output some next steps for setting up the API to use a bring-your-own-container setup. 

```
Next steps:
- Place your Dockerfile, docker-compose.yml and any related container source files in "amplify/backend/api/fastapirest/src"
- Amplify CLI infers many configuration settings from the "docker-compose.yaml" file. Learn more: docs.amplify.aws/cli/usage/containers
- Run "amplify push" to build and deploy your image
```

Following the above steps, the following **app/** and **Dockerfile** are added to the API directory in the Amplify backend.

```diff
  .
  ├── amplify/backend/api/<api-name>/src/
+ │   └── app
+ │       └── main.py
+ └────── Dockerfile

```

## Adding the Dockerfile

This can be modified based on the needs and requirements of the project but the below default Dockerfile will get you up-and-running immediately with FastAPI.  

I've adjusted the example in the FastAPI docs to use `python3.8` instead of `python3.7` **and** added the statement to expose port 80.


```Dockerfile
# Dockerfile
FROM tiangolo/uvicorn-gunicorn-fastapi:python3.8

# for Amplify 
# https://docs.amplify.aws/cli/usage/containers#deploy-a-single-container
EXPOSE 80

COPY ./app /app
```

{{< warning >}}

Make sure to add `EXPOSE 80` to specify a port to communicate with the container. Amplify will suggest to use port 80 if you don't provide one.

{{< /warning >}}


## Setting up the API 

Now, we can add in the API logic in the **main.py** file within **app/**. 


```python
# main.py

from typing import Optional
from fastapi import FastAPI

app = FastAPI()


@app.get("/")
def read_root():
    return {"Hello": "World"}


@app.get("/items/{item_id}")
def read_item(item_id: int, q: Optional[str] = None):
    return {"item_id": item_id, "q": q}

```


## Testing locally

With everything in place, we can test the API using Docker. Jump into the backend API directory (i.e. where the `Dockerfile` is) and build the image.

```
cd amplify/backend/api/<api-name>/src/
```

Build the image.

```
docker build -t fastapi .
```

Now, you can run the image locally, exposed on port 80 of your `localhost` setup.

```
docker run -d --name fastapi -p 80:80 fastapi
```

You'll see the auto-generated API documentation at [http://127.0.0.1/redoc](http://127.0.0.1/redoc) or [http://127.0.0.1/docs](http://127.0.0.1/docs).


## Deploying to Amplify 🚀

Awesome! Now let's push this to Amplify.

```
amplify push
```

When prompted, select `Yes` to **Create** the new resource.

```terminal
(🚀 dev) 🐶 ➜  fastapi-amplify amplify push                                                                                  (main) ✗
✔ Successfully pulled backend environment dev from the cloud.

Current Environment: dev

| Category | Resource name | Operation | Provider plugin   |
| -------- | ------------- | --------- | ----------------- |
| Api      | fastapirest   | Create    | awscloudformation |
? Are you sure you want to continue? Yes


```

After the process is complete. :point_down:

```
✔ All resources are updated in the cloud

REST API endpoint: https://<id>.execute-api.<region>.amazonaws.com
```


## Testing the deployed API

After being deployed, the API can be accessed using the endpoint above:


Also, the live API documentation is live at the `https://<endpoint>/docs` and `https://<endpoint>/redoc`.


### `/docs` route

{{< image src="images/docs.png" class="w-100 mh0 mb3" >}}

<br />


### `/redoc` route

{{< image src="images/redoc.png" class="w-100 mh0 mb3" >}}

<br />


This is just scratching the surface of configuration options. Hope this helps you get running quickly with serverless FastAPI backends!