+++
title = "Bring Your Own API Key: Supporting User-Provided OpenAI Keys and Prompts in Browser Extensions"
description = "Adding support for user-provided Gen AI LLM API keys in the dossi Chrome extension."

tags = [
    "generative AI",
    "LLM",
    "chrome extension",
    "github",
    "chat completion",
    "open source",
    "shadcn/ui",
    "open source maintainers",
]

# thumbnail = ""
# cardType = "summary_large_image"

date = 2024-06-24T01:34:21-04:00
categories = ["Development", "Open Source"]

+++

I recently added support (`v1.1.0+`) for bringing your OpenAI API to [dossi](https://dossi.dev) as an opt-in feature. This allows users to use their own OpenAI API key and prompts in the browser extension alongside parsed GitHub issue and discussion content. 

These are just a few thoughts on the implementation and some of my considerations when adding this in. 

## Overview

It was important to me that the addition strikes a balance between allowing the generative AI functionality and not being too prescriptive in how it’s used.

An example is in the video below. A user has enables the feature, adds their API key, and then adds a prompt. Then, the prompt is available to use when viewing a GitHub issue page and clicking the **Prompts** button. The prompt messages are composed of the user's input (previous notes) and the parsed GitHub issue. Then, the prompt is sent to the OpenAI API and the response is displayed in the extension. This response can be edited and saved as a new note.

{{< youtube q82eoIh1ecI >}}

<br />

In dossi, the Gen AI prompt functionality is a nice-to-have that can enhance the experience for users already using OpenAI for other projects. Or, for users who already use the OpenAI API.

Some common use cases for using prompts in dossi include:

- Generating summaries of GitHub issues and discussions
- Creating reproducible steps for bug reports
- Monitoring the sentiment of a conversation
- Determining contributors to an issue or discussion
- Staying up to date on the latest comments and discussions in a repository

To me, this can be a helpful tool for users who are managing multiple repositories, projects, or open-source organizations in GitHub. It can help them quickly get up to speed on the current state of an issue or discussion and help to reduce the overall time to close and triage for issues.


## Considerations for adding Gen AI

There are a few things to consider when adding this type of functionality. This isn't specific to dossi, but more of a general pattern that I've seen in other extensions that are backed by a hosted API and part of the reason why I decided to allow users to bring their own API key.

{{< note >}}

Note, that this is more geared towards SaaS-type browser extensions.

{{< /note >}}

Extensions backed by hosted API that integrates with LLM model providers...

- Pricing model: Typically a flat rate subscription fee, usage (credit) based, or both. This complicates the tracking of usage and billing for the user and couples the extension to the API provider.
- Abuse: Since the extension is a client-side application, users can abuse the API. This could be in the form of excessive requests or malicious content. Even if the user is subscribed to dossi, they may try to refute the charges after using the LLM functionality.
- User-defined input: Accepting, processing, and *relying* on arbitrary input (like GitHub issues) has a bunch of edge cases to account for (more on counting prompt input tokens below)
- Prompt response formatting: Everything works well until it doesn't. The response from the LLM model may not be what the user expects. This could be due to the prompt, the input, or the model itself.
- Handling errors without UX degradation

Allowing users to bring their own key eliminates most of the concerns. This also makes the functionality completely _opt-in_. The UX of the extension won't be impacted if a user doesn't want to use it.


## Adding Gen AI to dossi

Here's the high-level overview of the implementation in dossi:

{{< image src="images/dossi-llm.png" class="w-100 mh0 mb3" >}}


<br />

1. **Options Page** - Allow users to toggle on or off the use of their own API key. If enabled, a user can add their OpenAI API key and add in their own custom prompts. Once prompts exist, the UI button will be present when navigating GitHub issue and discussion pages.

2. **Local Storage** - In dossi, these settings are only saved on the local browser using the [`storage.local`](https://developer.chrome.com/docs/extensions/reference/api/storage#storage_areas). These settings are never sent to our servers or persisted in a remote database. This is different from the note, pin, and label data that is persisted in a hosted database.

3. **`Prompts` button** - The button is only visible when a user is on a GitHub issue or discussion page. The button is only visible if the user has enabled the feature, added an API key, and added at least one prompt.

4. **Gen AI** - The prompt is constructed using the user's input and the parsed GitHub issue or discussion. The prompt is sent to the OpenAI API and the response is displayed in the extension. The user can edit the response and save it as a new note.

5. **Error Handling** - Since this is the user's API key, I'm able to surface errors directly from the OpenAI SDK in the UI. Since the actions to configure the API key and prompts in dossi were explicit, the user has some awareness of the context of the error.

All of the messaging passes through the background service worker of the extension.



## Creating an Options Page

I decided to add an [options page](https://developer.chrome.com/docs/extensions/develop/ui/options-page) to the extension to make it easier and less cluttered to configure the local settings. Unlike the Chrome Side Panel API, an options page can be programmatically opened. This makes it convenient and more natural for users to navigate to. I added a settings link that triggers the page open in the extension user dropdown menus.


```javascript
// background.ts

chrome.runtime.onMessage.addListener(function (request, sender, sendResponse) {
  if (request.action === "openOptionsPage") {
    chrome.runtime.openOptionsPage()
  }
})

```
This is referenced in the user menu that is shared across the extension **popup** and **content script** side panel overlay.  



```tsx
// user-account-nav.tsx
<DropdownMenuItem
   onClick={() =>
     chrome.runtime.sendMessage({ action: "openOptionsPage" })
}>
  Settings
</DropdownMenuItem>
```

## Adding an API key and prompts

This options page provides a way for users to opt-in to API key use. Currently, if **opted-out**, then the side panel prompt button in the extension _will not be displayed_. This makes the functionality completely opt-in and will not clutter the UI if a user isn't interested in using it.

<br />

{{< image src="images/dossi-local-settings.png" class="w-100 mh0 mb3" >}}
<small class="gray">Options page in dossi</small>


If *enabled*, a user can add their API key *and* add in their own custom prompts. Once prompts exist, the UI button will be present when navigating GitHub issue and discussion pages. 

{{< youtube v0DEt2WxwLk >}}
<small class="gray">Adding a prompt in dossi local settings</small>

  
Since dossi is used with GitHub, a valid prompt for issues and discussion can be pretty broad. A few factors quickly come to mind:

* Is the user browsing, using, or maintaining the repo?
* What is the programming language used?
* Is the issue a bug, feature request, or question?
* Is subject matter expertise required?
* Is the /repo public or private?
* What is primary written language of the content?

I'd rather leave it up to the users to determine what information is the most useful for them.

## Saving configuration locally with Chrome Storage

{{< warning >}}

Always set spending limits on your API keys and monitor usage. The allowed permissions for the be as minimal as possible to reduce the risk of abuse.

{{< /warning >}}

Persisting data locally with Chrome storage is always a bit tricky. In dossi, these settings are only local.

This puts the control in the hands of the user. From a security perspective, there is no need for the extension or web app to store and manage user API keys on remote servers. If at any time a user wants to delete or invalidate their API key for use with dossi, they have at least 4 options:

1. Delete the API key at the provider level (i.e. OpenAI)
2. Clear all of their local dossi settings in the option page
3. Overwrite their stored API key with a new or invalid one
4. Remove the dossi extension (uninstall) which will wipe all of the associated storage data as per the Chrome API

Any of these actions can be done by the user, at any time. The note data, on the other hand, is remotely stored for a few reasons:

1. The data also powers the web app and additional integrations for users (i.e. reporting)
2. Storing large amounts of data locally may require additional browser extension permissions (`unlimitedData` )
3. Complex data models are difficult to troubleshoot, extend, rollback, and version - especially once published and in use

## Using the OpenAI JavaScript SDK

The [OpenAI JavaScript SDK](https://github.com/openai/openai-node) is used to interact with the OpenAI API. To use the functionality, users are required to create an OpenAI API key and grant `write` access for Model Capabilities and grant allowed models at the project level (i.e. `gpt-4o` ).  

{{< image src="images/model-capabilities.png" class="w-75 mh0 mb3 ba br1 b--black-20" >}}
<small class="gray">OpenAI API key settings</small>
<br />

For the initial implementation, users can define the core prompt `content`, `maxTokens`, and model (currently `gpt-4o`). For reference, here is the prompt schema using [`zod`](https://github.com/colinhacks/zod). Currently, the only available model is `gpt-4o` and provider is OpenAI.


```typescript
// This is the initial shape of the prompt
const promptSchema = z.object({
  title: z
    .string()
    .min(3, { message: "Title must be at least 3 chars" })
    .max(25, { message: "Title must be less than 25 chars" }),
  content: z
    .string()
    .min(25, { message: "Content must be at least 50 chars" })
    .max(1000, { message: "Content must be less than 1000 chars" }),
  model: z
    .string()
    .nonempty()
    .refine((m) => models.includes(m), {
      message: "Invalid model",
    }),
  maxTokens: z.coerce
    .number()
    .min(50, { message: "Max tokens must be between 50 and 1000" })
    .max(1000, { message: "Max tokens must be between 50 and 1000" }),
})
```

These parameters are passed along with **1)** the parsed page, and **2)** any previous dossi notes for the page (i.e. dossi entity) to include as context.

```typescript
const openai = new OpenAI({
  apiKey: apiKey,
  dangerouslyAllowBrowser: true,
})

// previous notes from user
const contextNotes = entity?.notes
  ?.map((note) => `${note.createdAt} ${note.content}`)
  .join("\n")

const promptInput = `
  ${prompt?.content}
  ---
  ${input}
`

const messages: ChatCompletionMessageParam[] = [
  {
    role: "system",
    content: "You are a helpful technical assistant.",
  },
  ...(contextNotes
    ? ([
        {
          role: "user",
          content: `Past notes for context: ${contextNotes}`,
        },
      ] as ChatCompletionMessageParam[])
    : []),
  { role: "user", content: promptInput },
]
```

I think in the future, including the previous user notes will be optional.  The `dangerouslyAllowBrowser` flag is set to `true` to allow the OpenAI SDK to be used in the browser. This is a requirement since the request is coming from the content script and not the background service worker.

```typescript
let chatCompletion: ChatCompletion

try {
  chatCompletion = await openai.chat.completions.create({
    messages: messages,
    model: prompt?.model,
    max_tokens: prompt?.maxTokens,
    n: 1,
    stop: null,
    temperature: 0.2,
  })
} catch (error) {
  console.error(error)
  // surface error to user
  ...
}
```
## Not hiding the implementation

By just providing the guardrails, I feel like it actually makes the use of the LLM functionality more tangible. The user shares control of the prompt input and output. Because of this, I feel like it gives the extension a bit more flexibility in what it does vs. the perceived "what it could do" of a completely hidden implementation. Also, the [extension is open source](https://github.com/siegerts/dossi-ext), so users can inspect the code and see how it works.

### Surfacing errors 

The first instance is presenting the user with errors. Since this is the user's API key and they own the configuration of that key, I'm able to surface errors directly from the OpenAI SDK in the UI. Since the actions to configure the API key and prompts in dossi are explicit, the user has some awareness of the context of the error.

{{< image src="images/surfaced-error.png" class="w-100 mh0 mb3" >}}
<small class="gray">Surfaced OpenAI API error in dossi</small>

I'm assuming that the user has some familiarity with the OpenAI API and can troubleshoot the error. If the error is related to the prompt, the user can edit the prompt and try again. If the error is related to the API key, the user can check the key and try again.

### Editing the prompt response

Secondly, the user can edit the prompt response directly in the extension and save it as a new note. This is a nice way to allow the user to review and update the response and make it more useful for their needs. Also, it's a nice way to smooth over any errors that may have occurred in the initial response.

{{< image src="images/review-prompt-response.png" class="w-100 mh0 mb3" >}}

## Flexibility on edge cases

There are a lot of edge cases when manually constructing prompts and including arbitrary (and unknown) input context in the form of parsed GitHub issues. The first is model token limitations. Models have constraints on the amount of input tokens, output tokens, and sometimes a combination. One approach to account for this is to count the prompt tokens before sending them to the model to gauge whether the prompt will meet the criteria. _To be honest, this isn't on my radar to add_.

This is a challenge in dossi since a GitHub issue may be extremely long with historical data. Additionally, bundling the dependencies into the extension may cause the size to increase impacting the overall experience for a user. 

I've decided to allow these types of errors to surface to the user instead of trying to intelligently truncate the context which could change the overall intent. Also, since this is the user's API key, I don't want to trigger additional calls to the completion APIs (i.e. chunk and summarize) without their explicit actions since there will be associated costs.

I may add a feature in the future to allow the user to manually truncate the input context if they are running into token limitations. Or, at a minimum, allow the parsed GitHub content to be viewed in the extension before sending the prompt to the model.

## Moving forward

Hope that you found this interesting! This is just the first iteration of the generative AI functionality in dossi. I'm excited to see how it's used.

The [browser extension and the web app are both open source](/post/dossi-is-now-open-source/). Check them out on GitHub:

- [Browser extension](https://github.com/siegerts/dossi-ext)
- [Web app and API](https://github.com/siegerts/dossi-app)

The extension is available in the [Chrome Web Store](https://chromewebstore.google.com/detail/ogpcmecajeghflaaaennkmknfpeghffm).