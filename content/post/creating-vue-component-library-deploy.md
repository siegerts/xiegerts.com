+++
title = "Deploy VuePress on Netlify"
description = "Documentation is no fun if it isn't public. Now, having configured the component library to use VuePress for the documentation and marketing aspect, we need to deploy it. Netlify is a great choice for this!"

tags = [
  "development", 
  "VuePress", 
  "Netlify",
  "static site",
  "documentation", 
  "deploy"]

categories = ["Development"]

series = "Vue Component Library"


date = 2019-05-23T21:19:10-04:00
+++

Documentation is no fun if it isn't public. Now, having configured the component library to use VuePress for the documentation and marketing aspect, we need to deploy it. [Netlify](https://www.netlify.com/) is a great choice for this! The VuePress documentation does a great job of documenting [deployment options](https://v1.vuepress.vuejs.org/guide/deploy.html#netlify). We'll use Netlify for this example.

## Deploy on Netlify

After the [last post](/post/creating-vue-component-library-documentation/), the component library plugin structure should be like the structure below. If you've modified some of the naming conventions, that's okay.

```diff
.
└─ docs/
  ├── .vuepress/
  │   ├─ components/
  │   │  ├─ examples/
  │   │  │  └─ standard-component-doc.vue
  │   │  ├─ Demo.vue
  │   │  └─ SourceCode.vue
  │   ├─ config.js
  │   └─ enhanceApp.js
  ├─ components/
  │  ├─ README.md
  │  └─ standard-component.md
  ├─ guide.md
  └─ README.md
```

You can link your account to the correct public repo and have the site build on a project `push`. This works really well if you're using GitHub, GitLab, etc.

Depending on your workflow, the build triggers can be configured if the generic setup needs to be modified.

Perfect. The documentation site is not live on the URL provided assigned by Netlify. :cocktail:

<div class="w-100 tc pv3 ba b--moon-gray">

<a href="https://vue-component-library-template.netlify.com/" target="_blank">https://vue-component-library-template.netlify.com/</a>

</div>

### Set up a custom domain

What if a custom domain better fits this project? Let's add it.

Create a `_redirects` file `.vuepress/public` for Netlify to pick up during the deploy process. Any files placed in the [public directory](https://v1.vuepress.vuejs.org/guide/assets.html#public-files) are copied to the _root_ of the generated directory when built.

```diff
.
└─ docs/
  ├── .vuepress/
  │   ├─ components/
  │   │  ├─ examples/
  │   │  │  └─ standard-component-doc.vue
  │   │  ├─ Demo.vue
  │   │  └─ SourceCode.vue
  │   ├─ config.js
  │   └─ enhanceApp.js
  ├─ components/
  │  ├─ README.md
  │  └─ standard-component.md
+ ├─ public/
+ │  └─ _redirects
  ├─ guide.md
  └─ README.md
```

The redirect information is available once the site is deployed and configured with a custom domain. Grab that configuration and add it to the new `_redirects` file. The information is in the _Domain management_ section in the Netlify console.

{{< note >}}

The example below illustrates a site that has HTTPS enabled through Netlify. Note the `https://`.

{{< /note >}}

```sh
# Redirect default Netlify subdomain to primary domain
https://<your-site-name>.netlify.com/* https://www.<your-custom-domain>/:splat 301!
```

The redirect will take effect on the next `git push` to the repo.

### Additional options provided by Netlify

- Snippet Injection
- Asset Optimization
- Prerendering
- Deploy Notifications

### Additional considerations for VuePress

- Sitemap (helps when setting up Google Webmaster Tools)
- OpenGraph tags with front matter

## Next steps

[Publish to npm](/post/creating-vue-component-library-npm/)!


If you have any questions or feedback, feel free to reach out on [X @siegerts](https://x.com/siegerts).