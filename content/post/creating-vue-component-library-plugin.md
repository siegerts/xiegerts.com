+++
title = "Consolidating components into a Vue.js plugin"
description = "At this point, we have a structured approach for creating new Vue.js components and consolidating them into a single module export. Awesome! Next, we'll bundle the components into a plugin to be registered on a Vue instance."
tags = [
    "development", 
    "vue", 
    "component library", 
    "plugin",
    "javascript"
]

categories = ["Development"]
series = "Vue Component Library"

date = 2019-03-11T12:40:05-05:00

+++

At this point, we have a [structured approach](/post/creating-vue-component-library-structure/) for creating new Vue.js components and consolidating them into a single module export. Awesome! Next, we'll bundle the components into a plugin to be registered on a Vue instance.

Remember that the Vue CLI creates a `main.js` entry point file in the root of the `/src` directory during the project initialization. Usually, that's used as the entry point for a new Vue application. We'll modify this to create the plugin.

If you're landing on this post without reading the [series introduction](/post/creating-vue-component-library-introduction/), jump there first.

## `main.js` entry point

Starting off, let's remove the generated code. We'll replace with the differences below.

```diff
// main.js

- import Vue from 'vue'
- import App from './App.vue'

- Vue.config.productionTip = false

- new Vue({
-  render: h => h(App),
- }).$mount('#app')

+ import * as components from './components'
+
+ const ComponentLibary = {
+  install(Vue, options = {}) {
+    // components
+    for (const componentName in components) {
+      const component = components[componentName]
+
+      Vue.component(component.name, component)
+    }
+  }
+ }
+
+ export default ComponentLibary
+
+ if (typeof window !== 'undefined' && window.Vue) {
+  window.Vue.use(ComponentLibary)
+ }
```

The file should look like:

```js
// main.js

import * as components from './components'

const ComponentLibary = {
  install(Vue, options = {}) {
    // components
    for (const componentName in components) {
      const component = components[componentName]

      Vue.component(component.name, component)
    }
  }
}

export default ComponentLibary

if (typeof window !== 'undefined' && window.Vue) {
  window.Vue.use(ComponentLibary)
}
```

Let's step through this :eyes:.

- Import the components from `src/components`. This will grab the components from the _exports_ in `index.js`. That's the file that imports(collects) the components that we want to include in the library.

* Now, we'll create the Vue plugin and expose an `install` method. According to the Vue [plugin documentation](https://vuejs.org/v2/guide/plugins.html#Writing-a-Plugin):

> _A Vue.js plugin should expose an install method...The method will be called with the Vue constructor as the first argument, along with possible options_

- In the `install` method, iterate through the imported components and assign each component to the `const component`. The `componentName` is used as a key get the component out of the `components` object.

- Register each component with `Vue.component()`. The `component.name` is the name _attribute_ from the component and the `component` as the component. When the plugin is registered in a Vue project, our components will be [globally available](https://vuejs.org/v2/guide/components-registration.html#Global-Registration).

{{< warning >}}
In the above, `componentName` is _not_ the same as `component.name`.
{{</ warning >}}

- Export the component library plugin as the default. This allows for importing into another file as `import ComponentLibrary from ...` syntax:

```js
import Vue from 'vue'
import ComponentLibrary from './main.js'

Vue.use(ComponentLibrary)

...
```

- Lastly, auto-register the plugin in situations where a Vue instance exists in the window and a module system is not used. We'll give this a test when we publish the library to a content delivery network (CDN) and include it on a page after the included Vue tag. This is covered in the Vue [Getting Started Guide](https://vuejs.org/v2/guide/index.html#Getting-Started) and is an option when adopting Vue into an existing application that may not use a build system.

Currently, the above set up does one thing - registering components. That's all we need it to do now but there are different patterns for plugin creation and the library entry point, `main.js`, in this case.

A few examples include:

- Adding directives, filters, and mixins
- Adding instance properties with `Vue.prototype`
- Importing style dependencies
- Merge user-defined options into the plugin registration with `options = {}`

The outline prescribed in the Vue documentation for writing a plugin is:

```js
// 1. add global method or property
Vue.myGlobalMethod = function () {
  // some logic ...
}

// 2. add a global asset
Vue.directive('my-directive', {
  bind (el, binding, vnode, oldVnode) {
    // some logic ...
  }
  ...
})

// 3. inject some component options
Vue.mixin({
  created: function () {
    // some logic ...
  }
  ...
})

// 4. add an instance method
Vue.prototype.$myMethod = function (methodOptions) {
  // some logic ...
}
```

<small class="link black-40">Source: [https://vuejs.org/v2/guide/plugins.html#Writing-a-Plugin](https://vuejs.org/v2/guide/plugins.html#Writing-a-Plugin)</small>

## One last tip

Also, always remember to populate a `name` attribute in your components if using the `Vue.component` and `component.name` approach above. The registration will throw an error if `component.name` doesn't exist.

```html
<template>
  <div>
    <slot></slot>
  </div>
</template>

<script>
  export default {
    name: 'name-of-your-component' // :point_left:
  }
</script>

<style></style>
```

<small class="black-40">Give your components a name :point_up:</small>

Next up, tightening the feedback loop plus a hint of amazing documentation with VuePress :volcano:.
