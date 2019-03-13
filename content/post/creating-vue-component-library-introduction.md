+++
title = "Creating a Vue.js component library: Introduction"
description = "In this series, we’ll focus on structuring a Vue component library as a plugin for use, and reuse. That also includes distribution and documentation. That's what is so great about Vue - the ability create your own building blocks for designing a user experience."
tags = [
    "development", 
    "vue", 
    "component library",
    "plugin",
    "vue cli 3", 
    "vuepress",
    "npm", 
    "javascript"
]
categories = ["Development"]

series = "Vue component library"

date = 2019-02-27T10:03:55-05:00
+++

In this series, we’ll focus on structuring a Vue component library as a plugin for use, and _reuse_. That also includes distribution and documentation. That's what is so great about Vue - the ability to create your own building blocks for designing a user experience. This series of posts _is not_ about writing components, that's a subject for another day.

## Context

I like to use existing component libraries until I don’t 😉.

There are many great libraries that already exist in the Vue ecosystem. Adhering to one _theme_ or _design_ system only works for so long, especially if you’re adopting Vue into an enterprise environment. In that case, there is usually a need for some consistency between interfaces (API & UI), style, and UX. For that reason, it’s important to know the basics of setting up your own library.

The information in the next few posts is not earth-shattering, but it’s all in one place. I’ve gone through some of the work of figuring out how the pieces fit together. This is not the end, it is a means to understanding the process to find your _own_ end.

{{< note >}}

As an aside, while thinking through this project (and others), I've come around to the idea of [renderless](https://adamwathan.me/renderless-components-in-vuejs/) components for reuse. This series will not focus building out a generic _renderless_ component library but it's worth a read if you're not familiar.

{{< /note >}}

## Workflow

I like quick iteration. We’ll focus on whipping up an environment that lends itself to quick visual feedback :rocket:. Once set up, you’ll be able to conceptualize a component, write it, register it, and document the specs. All that, without getting too bogged down in the nitty-gritty.

## Series agenda

I suppose you'll want to build your own components! Or, you already are?! Now you'll need to document, distribute, and manage enhancements. The structure outlined in the next few posts will help you achieve that goal.

We'll step through:

- Structuring a component library with [Vue CLI 3](https://cli.vuejs.org/guide/)
- Creating the plugin to register with Vue
- Documentation using [VuePress](https://vuepress.vuejs.org/)
- Publishing on [npm](https://www.npmjs.com/)
- Deploying the documentation

As always, the only way to understand is by rolling up your sleeves and giving it a try. :tada:

To get started, check out [Structuring a Vue component library](/post/creating-vue-component-library-structure/).
