+++
title = "Visual Studio Code settings for Vue"
description = "The settings below are my defaults for Vue.js development in Visual Studio Code."
tags = [
  "development", 
  "VS Code", 
  "Vetur",
  "Prettier",
  "eslint",
  "prettyhtml"
  ]
categories = ["Development"]

date = 2019-02-26T11:52:01-05:00

lastMod = 2019-04-09T11:52:01-05:00

[[resources]]
  name = "settings"
  src = "../images/vscode-vetur-prettier.png"
  title = "VS Code Vetur Prettier"
+++

The settings below are my defaults for Vue development in VS Code. These are optional _but_ influence the structure and syntax of the code examples in some of my posts. I've been using VS Code with Vue for a few years so I apologize if any of the settings are _legacy_. I try to keep it up to date. :calendar:

Extensions referenced here:

- [Vetur](https://marketplace.visualstudio.com/items?itemName=octref.vetur)
- [ESLint](https://marketplace.visualstudio.com/items?itemName=dbaeumer.vscode-eslint)
- [Prettier](https://marketplace.visualstudio.com/items?itemName=esbenp.prettier-vscode)
- editor (default)
- [Stylus Supremacy](https://marketplace.visualstudio.com/items?itemName=thisismanta.stylus-supremacy)

## `settings.json`

```json
{
  "vetur.format.styleInitialIndent": true,
  "vetur.format.scriptInitialIndent": true,
  "eslint.autoFixOnSave": true,
  "eslint.validate": [
    "javascript",
    "javascriptreact",
    {
      "language": "vue",
      "autoFix": true
    }
  ],
  "prettier.disableLanguages": [],
  "editor.formatOnSave": true,
  "prettier.eslintIntegration": true,
  "vetur.format.defaultFormatter.html": "prettyhtml",
  "vetur.format.defaultFormatterOptions": {
    "js-beautify-html": {},
    "prettyhtml": {
      "printWidth": 80,
      "singleQuote": false
    }
  },
  "stylusSupremacy.insertColons": false,
  "stylusSupremacy.insertLeadingZeroBeforeFraction": false,
  "stylusSupremacy.insertSemicolons": false,
  "stylusSupremacy.insertBraces": false
}
```

## Vetur extension config

Vetur defaults to Prettier for most options. I like to double check that the `<script>` tag is formatted by `prettier` and the `<template>` by [`prettyhtml`](https://prettyhtml.netlify.com/). The formatting can be a bit strange at first but the consistency becomes very helpful over time (at least for me).

{{< image src="images/vscode-vetur-prettier.png" class="w-100 w-70-ns mh0" >}}

## `.prettierrc.js`

These are a few of the stylistic preferences that I apply with Prettier. To use this [configuration](https://prettier.io/docs/en/configuration.html) approach, pop a `.prettierrc.js` file in your project root.

```js
// my preference
module.exports = {
  semi: false,
  singleQuote: true
}
```

## `.eslintrc.js`

These [ESLint](https://eslint.org/) settings coincide with the settings above. The Vue CLI generates this file on project initialization.

```js
module.exports = {
  root: true,
  env: {
    node: true
  },
  extends: ['plugin:vue/essential', '@vue/prettier'],
  rules: {
    'no-console': process.env.NODE_ENV === 'production' ? 'error' : 'off',
    'no-debugger': process.env.NODE_ENV === 'production' ? 'error' : 'off'
  },
  parserOptions: {
    parser: 'babel-eslint'
  }
}
```
