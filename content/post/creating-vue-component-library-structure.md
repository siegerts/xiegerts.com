+++
title = "Structuring a Vue component library"
description = "In this post, we'll step through the initial project structure for your Vue components. Remember, you'll want to to try to lower the barriers to entry of understanding for other developers in order for them to grasp your design choices. Not only for using, but for debugging (which is probably more likely, TBH)."
tags = [
    "development", 
    "vue", 
    "component library",
    "vue cli 3",
    "javascript"
]

categories = ["Development"]

series = "Vue component library"

date = 2019-02-28T12:38:50-05:00
+++

## Overview

In [the last post](/post/creating-vue-component-library-introduction/) I outlined why it is important to be able to package together a component library. In this post, we'll step through the initial project structure for your Vue components.

## Vue project structure

We're going to use the [Vue CLI 3](https://cli.vuejs.org/guide/). Bam!

Luckily, a lot of the _once was_ configuration steps are now handled by the CLI, mostly regarding webpack. That's not to say that you won't eventually need to [modify the webpack config](https://cli.vuejs.org/guide/webpack.html#simple-configuration) with `vue.config.js` but you'll be surprised how far you can get without doing that. I try to avoid modifying the generic webpack settings, if possible :pray:. Remember, you'll want to to try to lower the barriers to entry of understanding for other developers in order for them to grasp your design choices. Not only for using, but for debugging (which is probably more likely, TBH).

With that in mind, create your Vue project scaffold using the CLI.

```bash
vue create vue-component-library
```

After the project is created, and dependencies downloaded, you should see this in your terminal:

```bash

🎉  Successfully created project vue-component-library.
👉  Get started with the following commands:

 $ cd vue-component-library
 $ yarn serve

```

`vue-component-library` is the name of the component library project (folder, etc.). This _does not_ need to be the same as the programmatic representation of the library. We'll get into that in the upcoming _plugin_ post of the series.

When prompted during the project initialization, I choose the below options:

```
Vue CLI v3.0.0
? Please pick a preset: Manually select features
? Check the features needed for your project: Babel, Linter
? Pick a linter / formatter config: Prettier
? Pick additional lint features: Lint on save
? Where do you prefer placing config for Babel, PostCSS, ESLint, etc.? In dedicated config files
? Save this as a preset for future projects? (y/N) n
```

{{< note >}}
Make sure to adjust these options in the future if your preferences change. The Vue CLI comes bundled with a nifty [GUI](https://cli.vuejs.org/guide/creating-a-project.html#using-the-gui) that makes it easy to add and remove plugins.
{{< /note >}}

By default, the CLI with create the `scr/components` directory. I consolidate this project directory and project by removing unused items such as `App.vue`, `assets/favicon.ico`, etc. The initial structure is typically used as an application baseline. For a typical web application, I leave the setup as-is. Instead, we'll use VuePress for the documentation site functionality.

Next, we'll:

1. Remove the `public` directory
2. Remove `src/assets`
3. Remove `components/HelloWorld.vue`
4. Remove `src/App.vue`

The directory changes are diffed in the layout below.

```diff
  .
- ├── public/
  ├── src/
- │   ├─ assets/
  │   └─ components/
- │      └─ HelloWorld.vue
- └─ App.vue
```

You make be thinking..._did we just delete the whole project?_ Nope! The CLI adds a tremendous amount of functionality to your project besides the file layout. Note, `vue-cli-service` and the corresponding `devDependencies` in the generated `package.json`.

{{< tip >}}
Consider using the above generated view structure as a custom Vue app _or_ [ejecting your VuePress theme](https://vuepress.vuejs.org/default-theme-config/#ejecting) if you'd like less guardrails.
{{< /tip >}}

```json
{
  "name": "vue-component-library",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "serve": "vue-cli-service serve",
    "build": "vue-cli-service build",
    "lint": "vue-cli-service lint"
  },
  "dependencies": {
    "vue": "^2.6.6"
  },
  "devDependencies": {
    "@vue/cli-plugin-babel": "^3.0.0",
    "@vue/cli-plugin-eslint": "^3.0.0",
    "@vue/cli-service": "^3.0.0",
    "@vue/eslint-config-prettier": "^4.0.1",
    "babel-eslint": "^10.0.1",
    "eslint": "^5.8.0",
    "eslint-plugin-vue": "^5.0.0",
    "vue-template-compiler": "^2.5.21"
  }
}
```

<small class="black-40">Package versions may not be exactly the same depending on creation date.</small>

### Component directory structure

For each component, I create three files in a named component directory within `src/components`.

A generic view of the directory structure is:

```
.
└─ src/
  └─ components/
    └─ <component-identifier>/
      ├─ <component-identifier>.vue
      ├─ <component-identifier>.spec.js
      └─ index.js
```

Now, instead, for a hypothetical `Button` component:

```
.
└─ src/
  └─ components/
    └─ Button/
      ├─ Button.vue
      ├─ Button.spec.js
      └─ index.js
```

- `<component>.vue`

> Contains the single file component (SFC).

- `index.js`

> Imports and exports the component from the self-contained component directory.

- `<component>.spec.js`

> Where the component tests will live. If the tests create a snapshot, then a `__snapshots__` directory will be created within this directory.

So, for each of the files, let's create a placeholder.

#### `<component>.vue`

```html
<template>
  <div>
    <slot></slot>
  </div>
</template>

<script>
  export default {
    name: 'name-of-your-component'
  }
</script>

<style></style>
```

{{< tip >}}
Components do not need to end up completely self-contained (template + script + style) but I like to start with this approach. I refactor, if needed, as the library grows in size or complexity. There are a lot of opinions about (styles with JS) || (CSS in JS). I like to start with a plain ol' SFC + scoped styles and iterate from there.
{{< /tip >}}

Notice that the component has a `name`. This is very important and will impact our registering the library as a plugin in a few steps. Components are registered and referred to by the `name` attribute. Try to use an identifier that wont collide with other project dependencies or tags.

#### `index.js`

```js
import Button from './Button.vue'

export default Button
```

#### `<component>.spec.js`

We'll leave this file empty for now. Ultimately, this will contain the component tests.

#### Component export

Within in the `src` directory, create another `index.js` file to export the component(s). This file will sit alongside the top level `/components` directory as below.

```diff
 .
 └─ src/
   ├─ components/
   │  └─ ...
+  └─ index.js
```

In this file, we'll import, and export, the components from this file.

```js
// index.js
export { default as Button } from './Button'
```

This pattern may seem a bit repetitive but it provides flexibility in the library. The intermediate `index.js` file consolidates the components to be imported as a one-liner in the entry point file, _main.js_.

<div class="pa2 mv3 ba b--black-20">
<div class="flex items-center justify-center mv3">
<div class="flex-column dib ba b--black-20 b--dotted ph2 pv1">
  <span class="code f7">src/components</span>
  <div class="code f7 mv3 ba b--black-20 ph2 pv1 shadow-4">
    ComponentA
  </div>
  <div class="code f7 mv3 ba b--black-20 ph2 pv1 shadow-4">
     ComponentB
  </div>
  <div class="code f7 mv3 ba b--black-20 ph2 pv1 shadow-4">
     ComponentC
  </div>
</div>

  <div class="dib w-20 bb b--black-20 mv2"></div>
  <div class="dib pa1 bg-near-black br-100 nr1"></div>
  <div class="code f7 mv3 dib ba b--black-20 ph2 pv1 shadow-4">
    src/index.js
  </div>

  <div class="dib pa1 bg-near-black br-100 nl1"></div>
  <div class="dib w-20 bb b--black-20 mv2"></div>
  <div class="code f7 mv3 dib ba b--black-20 ph2 pv1 shadow-4">
    Vue plugin
  </div>
</div>
</div>
<small class="black-40">The components are exported, then imported/exported in the `src/index.js` file</small>

More than one component can live in the same `<component>` directory. For example, it may make sense to group components in a logical fashion based on usage pattern (i.e. `<List>` and `<ListItem>`). If so, adjust the above files to reflect:

```js
// src/components
import List from './List.vue'
import ListItem from './ListItem.vue'

export default { List, ListItem }
```

And one level higher:

```js
// src/index.js
export { List, ListItem } from './ListComponents'
```

The foundation is now set to add on the documentation part of the library :book:. We'll get to that next!
