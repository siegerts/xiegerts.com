+++
title = "Using VuePress for plugin documentation"
description = "Documentation time! In this post, we'll add a documentation element to the Vue component library plugin using VuePress. The end state of this post will be a static site that is structured to document a component library that exists in the same project. The component will generate its own documentation!"
tags = [
    "documentation", 
    "component library",
    "static site", 
    "vuepress",  
    "javascript"
]

categories = ["Development"]
series = "Vue Component Library"
date = 2019-03-21T12:37:21-05:00

[[resources]]
  name = "homepage"
  src = "../images/vuepress-home-page.png"
  title = "Initial VuePress home page"

[[resources]]
  name = "standard-component-route"
  src = "../images/standard-component-route.png"
  title = "standard-component-route"

[[resources]]
  name = "components-route"
  src = "../images/components-route.png"
  title = "components-route"
+++

Documentation time!

In this post, we'll add a documentation element to the Vue component library plugin using [VuePress](https://v1.vuepress.vuejs.org/). The end state of this post will be a static site that is structured to document a component library that _exists_ in the same project.

The component will generate its own documentation!

JavaScript is unique in that it's possible to create live documentation sites with all the available tools and frameworks. VuePress adds some extra icing :cake:. I've been surprised by how many documentation sites I've stumbled upon that use VuePress.

So, what's in the project already?

First, we created a few skeleton components as placeholders:

- [StandardComponent.vue](https://github.com/siegerts/vue-component-library-template/blob/master/src/components/StandardComponent/StandardComponent.vue)

- [FunctionalComponent.vue](https://github.com/siegerts/vue-component-library-template/blob/master/src/components/FunctionalComponent/FunctionalComponent.vue)

Then, we consolidated those into a Vue [plugin](/post/creating-vue-component-library-plugin/) in the last post. For reference, the source code for this post series is [here](https://github.com/siegerts/vue-component-library-template).

{{< note >}}
If you haven't been following along in the series then I encourage you to jump back to the [introduction](/post/creating-vue-component-library-introduction/) and start from there. That will provide a better context for the content of this post.
{{< /note >}}

Grab a cup of coffee (or tea) and let's get moving.

## Goals

The requirements for the documentation site include:

:white_check_mark: Display live component examples
\
:white_check_mark: The component is in the same project as the documentation
\
:white_check_mark: Changes are immediately reflected in documentation during development (i.e. hot-reload)
\
:birthday: VuePress provides all the elements of a featured static site

### Steps to achieve requirements

1. Add VuePress into the project
2. Customize `config.js` for our site layout and project metadata
3. Register the component library plugin _with_ the VuePress documentation site
4. Create a structure to visualize and document the components in the plugin

## Add VuePress

To begin, read through the [getting started](https://v1.vuepress.vuejs.org/guide/getting-started.html#inside-an-existing-project) part of the documentation if you aren't familiar with VuePress. For our use, we'll be _adding VuePress into an existing application_.

Following the documentation, let's add the latest VuePress version to our project.

```bash
yarn add -D vuepress@next
```

If you're following along in the [series](/series/vue-component-library/), then you should already have the `scripts` key in the `package.json`. This file is in the root of the project directory.

After installing VuePress, add the required commands:

```diff
...
"scripts": {
    "serve": "vue-cli-service serve",
    "build": "vue-cli-service build",
+   "docs:dev": "vuepress dev docs",
+   "docs:build": "vuepress build docs",
    "lint": "vue-cli-service lint"
  }
}
...
```

Next, remove Vue as a dependency using:

```sh
yarn remove vue
```

```diff
...

-"dependencies": {
-    "vue": "^2.6.6"
-  },

...
```

VuePress already has Vue as a [dependency](https://github.com/vuejs/vuepress/blob/master/packages/%40vuepress/core/package.json) so it isn't needed here to run or build the site. We'll add it as a _peer dependency_ for our plugin before we publish to npm.

We'll use `docs:dev` to develop and test our component library, and `docs:build` to build the documentation site for publishing (i.e. deployment).

Next, create a `docs` directory in the root of the project. The VuePress configuration and content will be placed in here.

```bash
# create a docs directory
mkdir docs
```

Within `docs`, create a `.vuepress` directory and also create a `README.md`. Make sure that both of these are in the `docs` directory.

Put the following YAML front matter in `README.md`:

```md
---
home: true
heroImage:
actionText: Get Started →
actionLink: /guide
features:
  - title: Feature 1
    details: Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.
  - title: Feature 2
    details: Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.
  - title: Feature 3
    details: Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.
footer: Vue Component Library 2019
---
```

This will become the documentation site homepage.

{{< danger >}}
The `README.md` file [needs to be present](https://v1.vuepress.vuejs.org/guide/getting-started.html#inside-an-existing-project) in the `docs` directory!
{{< /danger >}}

Also, add a `guide.md` file in `docs/`. We'll use this as a placeholder for a _Getting Started Guide_. Go ahead and put the following line in that file:

```md
# Getting Started

...
```

The project structure should look like:

```diff
.
└─ docs/
+ ├── .vuepress/
+ ├─ guide.md
+ └─ README.md
```

## Customize `config.js`

Following the VuePress documentation, let's customize the structure and settings for the site.

Add a `config.js` file in the `.vuepress` directory:

```diff
.
└─ docs/
  ├── .vuepress/
+ │   └─ config.js
  ├─ guide.md
  └─ README.md
```

This is a subset set of the [available options](https://v1.vuepress.vuejs.org/theme/default-theme-config.html#homepage). This template will be helpful as a starting point. Implementing all the available options here would be a bit overwhelming.

```js
// config.js

module.exports = {
  locales: {
    '/': {
      lang: 'en-US',
      title: 'Component Library 🥂',
      description: 'Documentation site for the Vue component library plugin'
    }
  },

  themeConfig: {
    repoLabel: 'Contribute!',
    // git repo here... gitlab, github
    repo: '',
    docsDir: 'docs',
    editLinks: true,
    docsBranch: 'dev',
    editLinkText: 'Help us improve this page!',
    search: false,
    locales: {
      '/': {
        label: 'English',
        selectText: 'Languages',
        lastUpdated: 'Last Updated',
        // service worker is configured but will only register in production
        serviceWorker: {
          updatePopup: {
            message: 'New content is available.',
            buttonText: 'Refresh'
          }
        },
        nav: [
          { text: 'Getting Started', link: '/guide' },
          { text: 'Components', link: '/components/' },
          // external link to git repo...again
          { text: 'GitHub', link: '' }
        ],
        sidebar: {
          '/components/': [
            {
              title: 'Components',
              collapsable: false,
              children: ['standard-component']
            }
          ]
        }
      }
    }
  }
}
```

Let's step through this a bit:

- Set the root locale as `en-US` with the appropriate site title.

- Add the `themeConfig`.

- The `nav` field takes a list of links that will be present along the top navigation of the site. The first link will point to `/guide` which displays the `guide.md` file that we created.

- The second link in `nav` points to `/components/` directory in `/.vuepress` that will contain the markdown files that document each component.

- The last link points to an external link, the GitHub repo link.

- Next, we add `sidebar`. In here, the above `/components` route is referenced. When that route is accessed, sidebar navigation will be present showing any available children routes.

- We'll add one child route in `sidenav` using `children: ['standard-component']`. `standard-component` refers to the name of the markdown files in the components directory. So, `/components/standard-component.md` :point_right: `standard-component`. This markdown file is rendered as HTML when the `<root>/components/standard-component` route is accessed.

At this point, the site should be able to run and serve with the default pages. Let's make sure that it works.

```sh
yarn docs:dev

...

VuePress dev server listening at http://localhost:8080/
```

{{< image src="images/vuepress-home-page.png" class="w-100 mh0 ba b--moon-gray shadow-4" >}}

The `/components` route will display a `404` page for now. That's okay since we will fix this in the next sections.

Great, now let's add the component library plugin.

## Register the component plugin

We'll also want to create and modify `enhanceApp.js` in the same `.vuepress/` directory.

```diff
.
└─ docs/
  ├── .vuepress/
+ │   ├─ enhanceApp.js
  │   └─ config.js
  ├─ guide.md
  └─ README.md
```

We'll import the library plugin from the `./../../src/main.js` entry point and register as a plugin within the documentation components.

{{< tip >}}
Remember the [plugin](/post/creating-vue-component-library-plugin/) we created in the last post? We're using it here!
{{< /tip >}}

#### `enhanceApp.js`

This allows the plugin to be available within the site. The structure of the [enhancement file](https://v1.vuepress.vuejs.org/guide/basic-config.html#app-level-enhancements) makes it easy to make _app-level_ adjustments.

Other items that can be added here include:

- Additional Vue plugins
- Register global components, or
- Add additional router hooks

```js
// enhanceApp.js

import ComponentLibrary from './../../src/main.js'

export default ({ Vue, options, router, siteData }) => {
  Vue.use(ComponentLibrary)
}
```

This is our component plugin :point_up:!

{{< warning >}}
The `enhanceApp.js` override allows for extra functionality to be added into the application. In this context, the _application_ refers to the documentation site. The component library plugin is contained in the same base project but _is not_ the application.
{{< /warning >}}

The components are now globally available in the documentation site. Now, we need to build out the actual documentation pages for each.

This part gets a bit tricky, so stick with me here :muscle:.

## Visualize and document the components

The goal is to show live examples of each component in the library plugin alongside its source code.

To accomplish that, a few files are needed first.

1. Create an example file. This is a single file component (SFC) exhibiting the component in different states, with different `props`, etc. These files will be located in `.vuepress/components/examples`.

2. Create a markdown file in `/components` for each plugin component. These pages will become the HTML pages. In this file, we'll leverage two global _presentational_ components, `Demo.vue` and `SourceCode.vue`, to link each plugin component and example SFC together.

We are going to create two presentation-related components, `Demo.vue` and `SourceCode.vue`, for the documentation aspects of the site. These components _are not_ part of the Vue plugin but will be available for us to use to structure the site pages. We're going to take advantage of [global components](https://v1.vuepress.vuejs.org/guide/using-vue.html#using-components) in VuePress here.

<div class="pa2 mv3 ba b--black-20 overflow-x-auto">
<div class="flex items-center justify-around justify-center mv3">

  <div class="flex-column dib ba b--black-20 b--dotted ph2 pv1">
  
  <div class="f7 code dib ph2 pv1">
     docs/components/test-component.md
  </div>
  <div class="code f7 mv3 ba b--black-20 ph2 pv1 shadow-4">
    <span>Demo.vue</span>

  <div class="code f7 mv3 ba b--black-20 ph2 pv1 shadow-4 bg-washed-blue">
     docs/.vuepress/components/examples/test-component.vue
    </div>

  </div>
  
  <div class="code f7 mv3 ba b--black-20 ph2 pv1 shadow-4">
     <span>SourceCode.vue</span>

  <div class="code ba b--black-20 ph2 pv1 shadow-4 bg-washed-blue">
     ./src/components/test-component/test-component.vue
    </div>

  </div>
  
</div>

  <div class="dib w-20 bb b--black-20 mv2"></div>
  <div class="dib pa1 bg-near-black br-100 nr1"></div>

  <div class="f7 mv3 dib ba b--black-20 b--dotted ph2 pv1">
    HTML page
    <div class="code f7 mv3 ba b--black-20 ph2 pv1 shadow-4">
    test-component.html
    </div>
  </div>
</div>
</div>
<small class="black-40">Using a hypothetical example component, _test-component_.</small>

#### Demo.vue

This component will be included in the component's documentation markdown file, `./docs/components/*.md`. It will wrap the component that is to be documented and inject it into the markdown page. In this case, it'll be set up to wrap the Vue files containing the plugin component example.

#### SourceCode.vue

This component will wrap a `<slot></slot>` that imports a code snippet. For our use, the snippet will be the source code of the component that is being documented. To do this, VuePress has a nifty feature that allows [importing code snippets](https://v1.vuepress.vuejs.org/guide/markdown.html#import-code-snippets) that we'll use.

### Creating `Demo.vue`

We want to create a structure that allows us to render each component into its documentation page. That way, a live example of the component is shown alongside the documentation.

Add the `Demo.vue` component in the `.vuepress/components` directory:

```diff
.
└─ docs/
  ├── .vuepress/
+ │   ├─ components/
+ │   │  └─ Demo.vue
  │   ├─ config.js
  │   └─ enhanceApp.js
  ├─ guide.md
  └─ README.md
```

```html
<!-- Demo.vue -->
<template>
  <div>
    <component :is="componentName" />
  </div>
</template>

<script>
  export default {
    props: {
      componentName: { type: String, required: true }
    }
  }
</script>
```

This is a straightforward component that takes a component filename reference as a `prop` and renders it as a [dynamic component](https://vuejs.org/v2/guide/components.html#Dynamic-Components) using the special attribute `:is`.

### Creating `SourceCode.vue`

```diff
.
└─ docs/
  ├── .vuepress/
  │   ├─ components/
  │   │  ├─ Demo.vue
+ │   │  └─ SourceCode.vue
  │   ├─ config.js
  │   └─ enhanceApp.js
  ├─ guide.md
  └─ README.md
```

```html
<!-- SourceCode.vue -->
<template>
  <div>
    <slot></slot>
  </div>
</template>
```

### Adding the first documentation page

#### Example file

For the first component's documentation, create an `example` directory and a Vue component to display examples of the selected component from the plugin. In this example, create a `standard-component.vue` to demonstrate the standard component from earlier in the series:

- `StandardComponent.vue` with the name attribute `standard-component`

{{< tip >}}
As a refresher, the component looks like [this](https://github.com/siegerts/vue-component-library-template/blob/master/src/components/StandardComponent/StandardComponent.vue).
{{< /tip >}}

```diff
.
└─ docs/
  ├── .vuepress/
  │   ├─ components/
+ │   │  ├─ examples/
+ │   │  │  └─ standard-component-doc.vue
  │   │  ├─ Demo.vue
  │   │  └─ SourceCode.vue
  │   ├─ config.js
  │   └─ enhanceApp.js
  ├─ guide.md // refers to the `/guide` route
  └─ README.md // need to have this == homepage!
```

In that example file, put the following code that demonstrates `standard-component` with different `slot` content.

```html
<template>
  <div>
    <standard-component>
      This is slot content 1.
    </standard-component>

    <standard-component>
      This is slot content 2.
    </standard-component>
  </div>
</template>
```

#### Markdown route file

The last file needed is the markdown file to pull it all together. First, add a `components` directory in `docs/`. Then, add another `README.md` file to that directory as shown below. This necessary and will act as an index page for the `/components` route of the site.

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
+ ├─ components/
+ │  ├─ README.md
+ │  └─ standard-component.md
  ├─ guide.md
  └─ README.md
```

In the `README.md` file, add:

```md
# Components

This is the index page for all the documented components.
```

In `.vuepress/config.js`, the `/components/` route of the site links to this markdown file with the reference `children: ['standard-component']`.

```js
// config.js from earlier
...
sidebar: {
  '/components/': [
    {
      title: 'Components',
      collapsable: false,
      children: ['standard-component']
    }
  ]
}

...
```

This means that VuePress will look in the `docs/components` directory in the project root and match against the markdown file of the same name.

So, let's create the markdown file that will associate with the `components/standard-component` route.

Add the content below to `standard-component.md` in `docs/components`:

```md
# standard-component

Wow! This component is awesome.

## Example

<Demo componentName="examples-standard-component-doc" />

## Source Code

<SourceCode>
<<< @/src/components/StandardComponent/StandardComponent.vue
</SourceCode>

## slots

...

## props

...
```

The `components/standard-component.md` file becomes the `components/standard-component.html` route of the documentation site!

Refreshing the site will activate the `/components` and `/components/standard-component` routes:

{{< image src="images/components-route.png" class="w-100 mh0 ba b--moon-gray shadow-4" >}}

{{< image src="images/standard-component-route.png" class="w-100 mh0 ba b--moon-gray shadow-4" >}}

Notice anything? The markdown is using the `Demo.vue` and `SourceCode.vue` components from earlier to show the example file and source code!

- `<Demo componentName="examples-standard-component-doc" />`

  - Be mindful of the `componentName` prop here, `examples-standard-component`. VuePress needs the directory structure to be hyphenated relative to the `.vuepress/components/` directory for global components. So, `examples-standard-component-doc` is equivalent to the `.vuepress/components/examples/standard-component-doc.vue` path.

- `<<< @/src/components/StandardComponent/StandardComponent.vue`
  - This line injects the source code snippet into the default `slot` of the `SourceCode.vue` utility component.

## Conclusion

Wow, that escalated quickly :wink:. This is a general setup that can be repeated as new components are added to the plugin - add another example Vue file and a markdown component file. Now, when you make changes in development mode, the site will immediately reflect those changes.

In the next post, we'll deploy the new documentation site to Netlify. After that, we'll publish the component library plugin available for distribution by publishing on **npm**.

As always, please reach out if you have any questions or advice :dog:.
