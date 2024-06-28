+++
title = "Detecting GitHub Issue Transfers in Chrome Extensions"
description = "Detecting issue transfers and redirects in browser extensions using the chrome.webNavigation API and the server_redirect transition qualifier."

tags = [
  "github api", 
  "browser extension",
  "webNavigation", 
  "open source",
  "chromium"
]


date = 2024-06-25T10:33:55-05:00

categories = ["Development", "Browser Extensions"]

series = "Developing Browser Extensions"
+++


In GitHub issues, if an issue is transferred to another repository **or** if the issue is converted to a discussion (or vice versa), the URL reference will change. Still it's not always easy to track these changes. Tracking transfers is pretty tricky, even using the GitHub API. If you're listening for webhook events on issues, it is possible to catch the `transferred` event *but* you *won't know where the issue was transferred to*.

In [dossi](https://dossi.dev), the extension listens for these URL changes in the browser and stores the URL in the storage. When the URL changes, the extension checks if the URL is a redirect. If a redirect (transfer) is detected, the UI will notify the user.

An example of a redirect is when an issue is transferred to another repository in GitHub. Or, if the organization or repository name changes, the URL will change.

Here's a video of how this works:

{{< youtube TdlyfXe1VXE >}}


This is something that I wanted to account for after running into this issue when tracking issue transfers in the past. Mainly, these links get shared around and included in docs, Slack and Discord messages, and other places. If the issue is transferred, I didn't want to lose the reference to the issue. It's a nuanced edge case but it can cause some head-scratching when you're trying to track issues across repositories or organizations.

## How to track issue transfers (side quest)

I've solved this in the past with the GitHub API but that requires [warehousing of all of the issues](https://github.com/siegerts/contributor-metrics/tree/main) (typically from an organization) in order to catch the transfer event and then check for the existence of the issue in another repository. In short, you need to check for "duplicates" of the issue in other repositories. Here's an example of how I've queried for this in the past using the GitHub API and a warehouse of issue data:

```sql
-- SQL query to find duplicate issues in a database
-- username is the issue creator
SELECT *
FROM issues
WHERE
	username || created_at in(
		SELECT
			username || created_at AS id FROM issues
		GROUP BY
			created_at, username
		HAVING
			count(*) > 1)
ORDER BY
	created_at, title, updated_at;
```
<small>Reference: [**contributor-metrics** GitHub repository](https://github.com/siegerts/contributor-metrics/blob/93b703e0504f300e441f623b5a5ba3a3d62455f9/chalicelib/transfers.py#L35)
</small>

This will isolate issues that have been created at the same time by the same user - which is a good indicator that the issue has been transferred since the issue will appear in two different repositories. Then it's possible to ping the issues and check if the returned status is a `301` with a new location header.

## Tracking issue transfers in dossi

dossi uses the [`chrome.webNavigation`](https://developer.chrome.com/docs/extensions/reference/api/webNavigation) API to listen for URL changes in the browser and stores the URL in the storage. When the URL changes, the extension checks if the URL is a redirect. If it is, it stores the redirect URL in the storage. 

This is useful for tracking the URL changes in the browser and is available using the `chrome.webNavigation` API. To use this API, you'll need to add the `webNavigation` permission in the extension manifest.

```json
{
  "permissions": ["webNavigation"]
}
```

You can add handlers to trigger for specific URL patterns using the `originAndPathMatches` property in `chrome.webNavigation`. In dossi, I'm using this to check for changes in discussions, issues, pull requests, and repositories in GitHub.

Here's an example of how this is implemented in the background script of the extension:

```ts
// using the plasmohq/storage library
import { Storage } from "@plasmohq/storage"
const storage = new Storage()


type UrlMatch = {
  url: string
  pos: number
} | null

type Redirect = {
  to: string
  from: string
}


const patterns = [
  {
    originAndPathMatches: `^https://github\.com/[a-zA-Z0-9\-_]+/[a-zA-Z0-9\-_]+/discussions/[0-9]+$`,
  },
  {
    originAndPathMatches: `^https://github\.com/[a-zA-Z0-9\-_]+/[a-zA-Z0-9\-_]+/issues/[0-9]+$`,
  },
  {
    originAndPathMatches: `^https://github\.com/[a-zA-Z0-9\-_]+/[a-zA-Z0-9\-_]+/pulls/[0-9]+$`,
  },
  {
    originAndPathMatches: `^https://github\.com/[a-zA-Z0-9\-_]+/[a-zA-Z0-9\-_]+$`,
  },
  { originAndPathMatches: `^https://github\.com/[a-zA-Z0-9\-_]+$` },
]


patterns.forEach((pattern, pos) => {
  chrome.webNavigation.onBeforeNavigate.addListener(
    async (details) => {
      await storage.set("from", { url: details.url, pos } as UrlMatch)
    },
    { url: [pattern] }
  )
  chrome.webNavigation.onCommitted.addListener(
    async (details) => {
      await storage.remove("redirect")

      if (details.transitionQualifiers.includes("server_redirect")) {
        logger.log("server_redirect detected.")

        let from: UrlMatch | null = await storage.get<UrlMatch>("from")

        if (!from) {
          return
        }

        const to = { url: details.url, pos } as UrlMatch

        if (from && to && from?.url !== to?.url && from?.pos == to?.pos) {
          await storage.set("redirect", {
            from: from?.url,
            to: to?.url,
          } as Redirect)

          // remove from storage
          await storage.remove("from")
        }
      }
    },
    { url: [pattern] }
  )
})

```

1. The `patterns` array contains the URL patterns that the extension listens for.
2. The `chrome.webNavigation.onBeforeNavigate` event listener is triggered when the URL is about to change. The current URL is stored in the storage.
3. The `chrome.webNavigation.onCommitted` event listener is triggered when the URL has changed. If the URL has a `server_redirect` transition qualifier, the extension checks if the URL has changed and if the URL is the same as the URL stored in the `onBeforeNavigate` event listener. If the URL has changed but the pattern type is the same, the extension stores the redirect URL in the storage.
4. The UI then displays a notification to the user in the side panel overlay.

{{< note >}}

**Note**: The `server_redirect` transition qualifier is one or more redirects caused by HTTP headers sent from the server happened during the navigation.

{{< /note >}}


## Request lifecycle breakdown

The pattern index (`pos`) is used to map the URL pattern to the URL stored in the `onBeforeNavigate` event listener. This is used to check if the changed URL type is the same as the type of the `onCommitted` URL.

So, an example of a GitHub issue transfer will follow this sequence of events:

1. URL with a pattern for *GitHub issue fires in `onBeforeNavigate` event listener
2. URL with a pattern for GitHub issue fires in `onCommitted` event listener
3. URLs are not the same *but* the positional mapping is the same
4. `server_redirect` is detected
5. This indicates that the GitHub issue has been transferred

<small>*GitHub issue pattern: `^https://github\.com/[a-zA-Z0-9\-_]+/[a-zA-Z0-9\-_]+/issues/[0-9]+$`</small>



{{< image src="images/dossi-issue-transfers.png" class="w-100 mh0 mb3" >}}

<br />

## Surfacing the Redirects in the Extension

The extension uses the `chrome.storage` API to store the redirect URL. When the extension is opened, it checks if there is a redirect URL in the storage and displays a notification to the user in the side panel overlay.

{{< image src="images/dossi-review-transfer-notes.png" class="w-100 mh0 mb3" >}}

The user can choose to view the notes and transfer them to the correct entity (i.e. new issue/repo URL).

{{< image src="images/dossi-transfer-modal.png" class="w-100 mh0 mb3" >}}


## Wrapping up

If you only work across a few repositories then you may not hit this issue often. But if you're working across multiple repositories or organizations, this situation is likely to pop up. I wanted a way to capture these changes without having to query the GitHub API for every issue in every repository or track events via webhooks. This ended up being a nice in-between solution that works well client-side but also gives users the option to review the changes.

The [dossi browser extension and the web app are both open source](/post/dossi-is-now-open-source/). Check them out on GitHub:

- [Browser extension](https://github.com/siegerts/dossi-ext)
- [Web app and API](https://github.com/siegerts/dossi-app)

The extension is available in the [Chrome Web Store](https://chromewebstore.google.com/detail/ogpcmecajeghflaaaennkmknfpeghffm). If you have any questions or feedback, feel free to reach out on [X @siegerts](https://x.com/siegerts).
