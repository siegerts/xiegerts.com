+++
title = "Adding Netlify redirect rules for Hugo"
description = "The initial jump into the netlify.toml file is a bit daunting at first, especially if you only want to make a small change. Once your Hugo site it deployed on Netlify, you’ll want to redirect traffic pointed to the provided subdomain to your primary domain."
tags = [
  "development", 
  "netlify", 
  "redirect", 
  "hugo",
  "golang",
  "deployment configuration"
  ]
categories = ["Development"]

date = "2019-02-22T10:37:21-05:00"

[[resources]]
  name = "redirect"
  src = "../images/netlify-redirect-rule.png"
  title = "Netlify redirect rule"

+++

The initial jump into the `netlify.toml` file is a bit daunting at first, especially if you only want to make a small change. Once your Hugo site it deployed on [Netlify](https://www.netlify.com/), you'll want to redirect traffic pointed to the provided subdomain **to** your primary domain.

### netlify.toml

Create a `netlify.toml` file in the root directory of your Hugo site. This file can be set up to do [quite a bit](https://www.netlify.com/docs/netlify-toml-reference/). For our purposes, we'll only configure the provided _redirect_ from the deployment.

```diff
  .
  ├── archetypes/
  ├── content/
  ├── themes/
  ├── config.toml
+ └── netlify.toml
```

The options specified in this configuration are merged with the settings specified in your Netlify console for your project. So, with the file in place, you can now phase in and test options as you become more familiar with the options.

### Configuration

The redirect information is available once the site is deployed and configured with a custom domain. Grab that configuration and add it to the `[redirect]` field in the `netlify.toml` file. The information is located in the _Domain management_ section in the Netlify console.

{{< note >}}

The example below illustrates a site that has HTTPS enabled through Netlify. Note the `https://`.

{{< /note >}}

```sh
# Redirect default Netlify subdomain to primary domain
https://<your-site-name>.netlify.app/* https://www.<your-custom-domain>/:splat 301!
```

Modify the `netlify.toml`:

```toml
# netlify.toml

[[redirects]]
  from = "https://<your-site-name>.netlify.app/*"
  to = "https://www.<your-custom-domain>/:splat"
  status = 301
  force = true #COMMENT: ensure that we always redirect
```

The same pattern can be repeated if more than one redirect is needed in the future. For example:

```toml
# netlify.toml

[[redirects]]
  from = ...
  to = ...
  ...

[[redirects]]
  from = ...
  to = ...
  ...

```

### Deploy

After deploying, you'll see the below (or something similar) in the _Deploy Summary_ confirming the redirect rule.

{{< image src="images/netlify-redirect-rule.png" class="w-100 w-70-ns mh0 ba b--moon-gray" >}}

### Extending netlify.toml

The Hugo documentation provides a generic [configuration baseline](https://gohugo.io/hosting-and-deployment/hosting-on-netlify/). You can adopt a more robust deployment over time by modifying the `netlify.toml` configuration as needed. Give it a look - I'm sure that you'll find some interesting nuggets that will be helpful for your next project! :tada:
