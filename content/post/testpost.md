+++
title = "🐶 A big test"
description = ""
tags = [
   
    "development",
]
topics = [
    "Development",
    
]
date = 2019-02-01T17:05:44-05:00
+++

## Getting started

Page-level variables are defined in a content file’s front matter, derived from the content’s file location, or extracted from the content body itself.
The following is a list of page-level variables. Many of these will be defined in the front matter, derived from file location, or extracted from the content itself.

```html
<meta property="og:type" content="article" />
<meta property="og:description" content="{{ .Description }}" />
<meta property="og:title" content="{{ .Title }} : xiegert.com" />
<meta property="og:site_name" content="xiegert" />
<meta property="og:image" content="" />
<meta property="og:image:type" content="image/jpeg" />
<meta property="og:image:width" content="" />
<meta property="og:image:height" content="" />
<meta property="og:url" content="{{ .Permalink }}" />
<meta property="og:locale" content="{{ .Site.LanguageCode }}" />
<meta property="article:published_time" content="{{ .Date.Format "2006-01-02"}}
" /> <meta property="article:modified_time" content="{{ .Date.Format
"2006-01-02"}} " /> {{if .Keywords }} {{ range.Keywords }}<meta
  property="article:tag"
  content="{{ . }}"
/>
{{ end }} {{else if isset .Params "tags" }} {{ range.Params.tags }}<meta
  property="article:tag"
  content="{{ . }}"
/>
{{ end }} {{ end }}
```

`this is a test`

Page-level variables are defined in a content file’s front matter, derived `from the content’s file` location, or extracted from the content body itself.
The following is a list of page-level variables. Many of these will be defined in the front matter, derived from file location, or extracted from the content itself.
