+++
title = "Publishing a Vue component plugin on npm"
description = "The last step in creating a Vue component library plugin is to publish it as a package. Most packages are published on npm if the intention is to distribute to an external audience. Other registry options include GitHub Package Registry and Artifactory. It is also possible to run your own private registry. To demonstrate, we'll publish to npm. Similar steps can be taken to use a different registry if it also utilizes the npm (or yarn) CLI API conventions."

tags = [
   "development",
   "vue",
   "vue.js",
   "component library",
   "vue cli 3",
   "vuepress",
   "publish",
   "npm",
   "package",
   "package.json",
   "javascript"
]
 
categories = ["Development"]
series = "Vue Component Library"
date = 2020-01-04T12:38:20-05:00
+++

The last step in creating a Vue component library plugin is to publish it as a package. Most packages are published on npm if the intention is to distribute to an external audience. Other registry options include [GitHub Package Registry](https://github.com/features/packages) and [Artifactory](https://www.jfrog.com/confluence/display/RTF/Npm+Registry). It is also possible to run your own [private registry](https://docs.npmjs.com/misc/registry#can-i-run-my-own-private-registry).

In this post, I'll explain the process to publish to npm. Similar steps can be taken to use a different registry if it also utilizes the `npm` (or `yarn`) CLI API conventions.

## npm

First, create an account on [npm](https://www.npmjs.com) - and set up multi-factor authentication!

This is where you will need to choose your package name and permissions. It makes sense to make sure that the package name that you want, or coincides with your library's functionality, is available before solidifying the name within the library references itself.

{{< warning >}}

Check for package name availability before buying a domain name!

{{< /warning >}}

The `name` field in the `package.json` file will be used to determine the package name. So, do a bit of investigating on npm first to make sure that the package name **is available**.

{{< note >}}

A note on semantics: The Vue component _library_ will be published as a _package_ on npm.

{{< /note >}}

## Publishing

To publish our library, we'll need to make some additions to the `package.json` file.

I'll be using the `vue-example-pkg` as the name. Make sure to swap any references to your own package name when you see `vue-example-pkg`.

#### `package.json`

Below is an example `package.json` as a reference when crafting your own based on your package's functionality and assets. This file should look familiar - it's an extension of the same `package.json` file that's been used throughout the [series](/series/vue-component-library/).

A full reference of all available options with explanations is available in the [npm documentation](https://docs.npmjs.com/files/package.json).

```js
{
 // name of the library on npm!
 "name": "vue-example-pkg",
 "version": "0.1.0",
 // If you set "private": true in your package.json, then npm will refuse to publish it.
 "private": false,
 "main": "dist/vue-example-pkg.umd.min.js",

 // this makes sure that library is distributed to a CDN
 "unpkg": "dist/vue-example-pkg.umd.min.js",
 "jsdelivr": "dist/vue-example-pkg.umd.min.js",

 "author": "Your name",
 "license": "MIT", // or whatever you decide
 "description": "",
 "files": [
   "dist/*",
   "src/*"
 ],
 "homepage": "",
 "repository": {
   "type": "git",
   "url": "https://github.com/siegerts/vue-example-pkg.git"
 },
 "bugs": {
   "url": "https://github.com/siegerts/vue-example-pkg/issues"
 },
 "scripts": {
   "serve": "vue-cli-service serve",

   // tell Vue CLI that you want this project built as a library
   "build": "vue-cli-service build --target lib --name vue-example-pkg src/main.js",
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

#### `build`

- _tell the Vue CLI that you want this project built as a library_

It's important to review the [Vue CLI build target](https://cli.vuejs.org/guide/build-targets.html#build-targets). Specifically, we'll be building this package as a [Vue library](https://cli.vuejs.org/guide/build-targets.html#library). This will bundle the library in the same way that we've been referencing it previously but with the new package _name_ instead of the previous path reference.

```js
import ComponentLibrary from 'vue-example-pkg'

Vue.use(ComponentLibrary)
```

#### `prepublishOnly`

- _builds the library before publishing to npm; points to `build` script_

The above incantation is pretty bare-bones. Other commands can be run at this point such as _tests_ and _linting_ depending on your workflow. Just be aware that `prepublishOnly` is used as the last set of commands before publishing when running `yarn publish` (or `npm publish`).

It's important to note the `$npm_execpath` reference in this command. This is an environment variable that determines what _npm_ to use. This may sound a bit strange at first :smile:. It comes in handy when the `yarn` package manager is used instead of `npm`. This doesn't make assumptions about package manager to use and instead uses what is currently set (i.e invoked). If you're using Windows machine, then you will need to swap this out for `%npm_execpath%`.

For reference:

- https://docs.npmjs.com/misc/scripts
- https://stackoverflow.com/questions/43421829/how-to-dynamically-select-package-manager-in-package-json
- https://stackoverflow.com/a/51793644/2205847

#### Distributing to a CDN

The lines referencing the CDNs will chose a file to distribute, and make available on each CDN, respectively. This is nice if you want your package to be available to those not using a local package manager in their projects.

```js
"unpkg": "dist/vue-example-pkg.umd.min.js",
"jsdelivr": "dist/vue-example-pkg.umd.min.js",
```

For more information regarding [jsdelivr](https://www.jsdelivr.com/) setup:

- https://github.com/jsdelivr/jsdelivr#configuring-a-default-file-in-packagejson

## Wrapping up

Now running your publish command (below) should initialize the publishing process to npm.

```sh
$ yarn publish  # or npm publish
```

I'm a realist :innocent:. There _will_ be hang ups going through this process to get it exactly right for your project. That's okay. Remember to have fun. There are a lot of commands and options. The important thing is to get a solid foundation working and then iterate and tweak from that point :thumbsup:.

Hope that the series has been helpful! If so (or not!), let me know - [@siegerts](https://x.com/siegerts).

