+++
title = "Create a Vue.js component library: Vue CLI 3, VuePress, and npm"
description = "Structure, create, and publish a Vue.js component library on npm with Vue CLI 3 and VuePress"

tags = [
    "development", 
    "vue", 
    "vue.js", 
    "component library", 
    "vue cli 3", 
    "vuepress", 
    "publish", 
    "npm", 
    "javascript"
]

categories = ["Development"]

series = "Vue Component Library"

date = 2019-02-15T08:15:03-05:00

draft = true

+++

Writing the components is the easy part :smile:.
structure, create, and publish a Vue.js component library on npm with Vue CLI 3 and VuePress

## concept

## library structure

```bash
vue create componentLib
```

### src/components

#### components/<component>

`<component>.vue`

```html
<template functional>
  <button>
    <slot></slot>
  </button>
</template>

<script>
  export default {
    name: "nks-btn",
    ...
  }
</script>
```

`index.js`

```js
import Button from "./Button.vue";

export default Button;
```

`<component>.spec.js`

#### `index.js`

```js
export { default as Button } from "./Button";
export { default as Card } from "./Card";
...
```

### styles

### tests

### library plugin entry point

#### `main.js`

```js
import "tachyons/css/tachyons.min.css";
import * as components from "./components";
import config, { setOptions } from "./utils/config";

const Nooks = {
  install(Vue, options = {}) {
    // options
    setOptions(Object.assign(config, options));

    // components
    for (const componentName in components) {
      const component = components[componentName];

      if (component && componentName !== "install") {
        Vue.component(component.name, component);
      }
    }
  }
};

export default Nooks;

if (typeof window !== "undefined" && window.Vue) {
  window.Vue.use(Nooks);
}
```

### workflow

## documentation

```bash
yarn add -D vuepress
```

```bash
# create a docs directory
mkdir docs
```

```json
{
  "scripts": {
    "docs:dev": "vuepress dev docs",
    "docs:build": "vuepress build docs"
  }
}
```

### structure & configuration

- [ ] display live components
- [ ] component lives by the documentation (and tests)
- [ ] changes are immediately reflected in documentation

#### `config.js`

```js
module.exports = {
  plugins: [],
  // cache: false,
  locales: {
    "/": {
      lang: "en-US",
      title: "nooks",
      description: ""
    }
  },

  themeConfig: {
    repoLabel: "Contribute!",
    repo: "https://github.com/siegerts/nooks",
    docsDir: "docs",
    editLinks: true,
    docsBranch: "dev",
    editLinkText: "Help us improve this page!",
    search: false,
    locales: {
      "/": {
        label: "English",
        selectText: "Languages",
        lastUpdated: "Last Updated",
        serviceWorker: {
          updatePopup: {
            message: "New content is available.",
            buttonText: "Refresh"
          }
        },
        nav: [
          // { text: 'Nooks', link: '/' }
          { text: "Getting Started", link: "/guide" },
          { text: "Components", link: "/components/" },
          { text: "Github", link: "https://github.com/siegerts/nooks" }
        ],
        sidebar: {
          "/components/": [
            {
              title: "Components",
              collapsable: false,
              children: ["btn", "card", "node"]
            }
          ]
        }
      }
    }
  }
};
```

#### `enhanceApp.js`

```js
import Nooks from "../../src/main.js";

export default ({ Vue, options, router, siteData }) => {
  Vue.use(Nooks);
};
```

### sitemap

### og tags

## publishing

[unpkg](https://unpkg.com/)

#### `package.json`

```json
{
  // name of the library on npm!
  "name": "vue-nooks",
  "version": "0.1.0",
  "private": false,
  "main": "dist/nooks.umd.min.js",

  // this makes sure that library is distributed to a CDN
  "unpkg": "dist/nooks.umd.min.js",
  "jsdelivr": "dist/nooks.umd.min.js",

  "author": "Stephen Siegert",
  "license": "MIT",
  "description": "",
  "files": [
    "dist/*",
    "src/*"
  ],
  "homepage": "https://nooksjs.org",
  "repository": {
    "type": "git",
    "url": "https://github.com/siegerts/nooks.git"
  },
  "bugs": {
    "url": "https://github.com/siegerts/nooks/issues"
  },
  "scripts": {
    "serve": "vue-cli-service serve",

    // tell Vue CLI that you want this project built as a library
    "build": "vue-cli-service build --target lib --name nooks src/main.js",
    "lint": "vue-cli-service lint",

    // builds the library before publishing to npm; points to `build` script above
    "prepublishOnly": "$npm_execpath build",

    // builds documentation; for use with doc deploy (i.e. Netlify or other)
    "docs:dev": "vuepress dev docs",
    "docs:build": "vuepress build docs"
  },
  "dependencies": {
    ...
  },
  "devDependencies": {
    "@vue/cli-plugin-babel": "^3.3.0",
    "@vue/cli-plugin-eslint": "^3.3.0",
    "@vue/cli-service": "^3.3.0",
    "@vue/eslint-config-prettier": "^4.0.1",
    "babel-eslint": "^10.0.1",
    "eslint": "^5.8.0",
    "eslint-plugin-vue": "^5.0.0",
    "style-resources-loader": "^1.2.1",
    "stylus": "^0.54.5",
    "stylus-loader": "^3.0.2",
    "vue-cli-plugin-style-resources-loader": "^0.1.3",
    "vue-template-compiler": "^2.5.21",
    "vuepress": "^1.0.0-alpha.32"
    ...
  },
  "peerDependencies": {
    "vue": "^2.5.21"
  },

  // once again, SEO
  "keywords": [
    "vue",
    "vuejs"
    ...
  ]
}

```

## hosting

https://vuepress.vuejs.org/guide/deploy.html#netlify
