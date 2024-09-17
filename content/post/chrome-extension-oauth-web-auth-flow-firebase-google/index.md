+++
title = "Adding Sign In with Google OAuth to a Chrome Extension: chrome.identity.launchWebAuthFlow"

description = "How to set up Google Sign In with Google OAuth in a Chrome Extension using chrome.identity.launchWebAuthFlow to handle the OAuth flow across all Chromium-based browsers. This is the first post in a series on implementing Google Sign In using Google OAuth in Chrome extensions."

tags = [
    "chrome extension",
    "firebase",
    "oauth",
    "google oauth",
    "google sign in",
    "chromium",
    "brave",
    "chrome",
    "chrome.identity",
]

categories = ["Development"]
date = 2024-09-14T15:25:16-04:00

series = "Developing Browser Extensions"

+++

Google Auth in Chrome Extensions is fun :smirk:

If you're building OAuth into a Chrome Extension you'll likely start with `chrome.identity.getAuthToken` and then realize that it doesn't work in other browsers outside of Chrome, even Chromium-based ones like Brave. [`chrome.identity.launchWebAuthFlow`](https://developer.chrome.com/docs/extensions/reference/api/identity#method-launchWebAuthFlow) is the alternative but it's a bit more involved since you'll need to handle the token exchange and refresh yourself.

A few items that you may want to take into consideration. First, you can't use the same code to authenticate users in other browsers if you're already using `getAuthToken` and the second is that you you'll need to implement access token handling and refresh yourself if you chose to go the `launchWebAuthFlow` route. The Chrome API docs state that `chrome.identity.launchWebAuthFlow` is for non-Google OAuth providers but that's not very practical since it's the only way to get the access token in other browsers - we'll use it for Google Sign In as well.

{{< note >}}

I'm mostly documenting this so I don't need to relearn this next time I need to do it. This is not meant to be a tutorial but more of a reference for your own implementation and architecture.

{{< /note >}}

## Overview

This post assumes that you already have a Chrome extension (Manifest v3) configured to use Google Sign In with OAuth. And also that you've created a Google Cloud project with the necessary APIs enabled. I also use Firebase Auth to manage the user state in the extension but you can use any other method to manage the user state Additionally, I use [Plasmo with React](https://docs.plasmo.com/) but you should be able to swap in another framework or use with Vanilla JS since we'll just be using the Chrome API.

This might be a little tricky if you are new to Chrome extensions since there will be some messaging between the background script and client script that will trigger the OAuth flow. So, just be prepared to do some debugging.

I'll be breaking this down into three parts:

1. Configuring `chrome.identity.launchWebAuthFlow` for Google Sign In in the extension (this post)
2. [Setting up a Cloudflare Worker to handle the token exchange and refresh](/post/chrome-extension-google-oauth-access-token/)
3. [Handling the token revoke and refresh in the extension](/post/chrome-extension-google-oauth-refresh-token/)


We'll be doing the following:
- Allow users to initiate the OAuth flow in the extension using `chrome.identity.launchWebAuthFlow`
- Obtain the **authorization code** from the Google OAuth flow
- Exchange the **authorization code** for an **access token** and **refresh token**
- Store the **refresh token** in the Chrome storage
- Use the **access token** to authenticate with Google APIs
- Use the **refresh token** to get a new **access token** when the current one expires
- Listen for sign outs, token expiration, token refresh, and token revoke from the Google account


## Chrome Identity API browser compatibility

Now, should you use your own flow in combination with `chrome.identity.getAuthToken` for Chrome users since it's a stable API and then use your own flow for other browsers? Or should you just use your own flow using `chrome.idenity.launchWebAuthFlow` for all users? That's up to you and your requirements. Just know that, as of now, [`chrome.identity.getAuthToken` doesn't work in Brave](https://github.com/brave/brave-browser/issues/7693) and other Chromium-based browsers - the compatibility is not reliable.


`chrome.identity.launchWebAuthFlow` initiates the OAuth 2.0 flow in the Chrome extension and typically provides an authorization code or access token, depending on the OAuth provider's flow. Access tokens obtained via this flow have an expiration time, after which they will become invalid. Refresh tokens, if available, are not automatically managed by Chrome's `launchWebAuthFlow`. You need to explicitly manage them in your extension. Most write-ups on this topic will only cover the access token and not the refresh token. This is important because the access token will expire and you'll need the refresh token to get a new access token.

To handle the token revoke, you'll need to listen for sign outs, token expiration, token refresh, and token revoke from the Google account.

To handle these scenarios, we'll need to do a few things:

1. Allow users to initiate the OAuth flow in the extension using `chrome.identity.launchWebAuthFlow`
2. Obtain the **authorization code** from the Google OAuth flow
3. Exchange the **authorization code** for an **access token** and **refresh token**
4. Store the **refresh token** in the Chrome storage
5. Use the **access token** to authenticate with Google APIs
6. Use the **refresh token** to get a new **access token** when the current one expires
7. Listen for sign outs, token expiration, token refresh, and token revoke from the Google account

{{< warning >}}

Notice that this is the **authorization code** and not the **access token**. We'll need to exchange this code for an access token and refresh token. This is different from purely just signing a user in with Firebase. We need to sign in with Google first to get the access token to authenticate with Google APIs. We can't use the `idToken` that we get from Firebase to authenticate with Google APIs. So, this is only necessary if you need to access Google APIs after the user is signed in.

{{< /warning >}}

If you're using Firebase Auth then you can use the returned **access token** to authenticate with Firebase via `GoogleAuthProvider.credential` and `signInWithCredential`.

{{< note >}}

As a general rule, I use both a dev and prod environment for extensions that have external dependencies. This is because the extension will have different origins in development and production. This is important because the OAuth flow will require a redirect uri that is registered with the OAuth client in the Google Cloud Console. So, when setting this up, I actually use an OAuth client in the Google Cloud Console for the dev environment and another for the prod environment.  

{{< /note >}}


## Configuring `chrome.identity.launchWebAuthFlow` for Google Sign In in the extension

The first step is to allow users to initiate the OAuth flow in the extension using `chrome.identity.launchWebAuthFlow`. This will open a new window in the browser where the user can sign in with Google. Once the user signs in, they will be redirected back to the extension with an authorization code that we can exchange for an access token and refresh token.

```tsx
const onLaunchWebAuthFlow = async () => {
    try {
      const authUrl = new URL("https://accounts.google.com/o/oauth2/auth")
      const clientId = "<your-oauth-client-id>"

      // Note: this needs to match the one used on the server (below)
      // note the lack of a trailing slash
      const redirectUri = `https://${chrome.runtime.id}.chromiumapp.org`

      const state = Math.random().toString(36).substring(7)

      const scopes = "profile email <other scopes>"

      authUrl.searchParams.set("state", state)
      authUrl.searchParams.set("client_id", clientId)
      authUrl.searchParams.set("redirect_uri", redirectUri)

      authUrl.searchParams.set("scope", scopes)
      authUrl.searchParams.set("response_type", "code")
      authUrl.searchParams.set("access_type", "offline")
      authUrl.searchParams.set("include_granted_scopes", "true")
      authUrl.searchParams.set("prompt", "consent")

      chrome.identity.launchWebAuthFlow(
        {
          url: authUrl.href,
          interactive: true,
        },
        async (redirectUrl) => {
          if (chrome.runtime.lastError || !redirectUrl) {
            return new Error(
              `WebAuthFlow failed: ${chrome.runtime.lastError.message}`,
            )
          }

          const params = new URLSearchParams(redirectUrl.split("?")[1])
          const code = params.get("code")

          if (!code) {
            return new Error("No code found")
          }

          // exchange code for access token
          // in next series post
          console.log("code: ", code)

          } catch (error) {
            throw new Error(`OAuth Sign-in failed: ${error.message}`)
          }
        },
      )
    } catch (error) {
      throw new Error(`Sign-in failed: ${error.message}`)
    }
  }

```

Let's step through the code:

1. We create a new URL object with the Google OAuth URL
2. The `client_id` is the OAuth client ID that you get from the Google Cloud Console for the web client
3. The `redirect_uri` allows the OAuth flow to redirect back to the extension after the user signs in. There is a method, [`chrome.identity.getRedirectURL`](https://developer.chrome.com/docs/extensions/reference/api/identity#method-getRedirectURL), that you can use to get the redirect URL for the extension **but** this will include a trailing slash which will not be accepted as an authorized redirect URI in the Google Cloud Console. So, you'll need to manually construct the redirect URL without the trailing slash or remove if you use the method.
4. The `state` is a random string that is used to prevent CSRF attacks. This gets passed along with the OAuth flow and is returned back to the extension to verify the request.
5. The `scopes` are the permissions that you are requesting from the user. You can add more scopes as needed but they should match the ones that you have set up in the Google Cloud Console.
6. The `response_type` is set to `code` since we are expecting an authorization code from the OAuth flow. This is important because we need to exchange this code for an access token and refresh token. If you only get the access token then you won't be able to refresh it when it expires _which will require the user to sign in again_.
7. The `access_type` is set to [`offline`](https://developers.google.com/identity/protocols/oauth2/web-server#offline) so that we can get a refresh token along with the access token. This is important because we need the refresh token to get a new access token when the current one expires.
8. The `include_granted_scopes` is set to `true` so that we can get the scopes that the user has already granted.
9. The `prompt` is set to `consent` so that the user is prompted to grant the permissions that we are requesting. This is important because we need the user to grant the permissions that we are requesting. If this is not set then the user will not be prompted to grant the permissions and the OAuth flow will fail unless the user has already granted the permissions. Also, if the user has been signed out, the consent prompt will be shown again.
10. The flow is initiated with `chrome.identity.launchWebAuthFlow` and the `url` is set to the `authUrl` that we created. The `interactive` is set to `true` so that the user is prompted to sign in. The callback function will be called with the `redirectUrl` which will contain the authorization code that we need to exchange for an access token and refresh token.
11. The `redirectUrl` is parsed using `URLSearchParams` to get the authorization code which is then used to exchange for an access token and refresh token.


More information on the individual parameters can be found in the [Obtaining OAuth 2.0 access tokens](https://developers.google.com/identity/protocols/oauth2/javascript-implicit-flow#obtainingaccesstokens) documentation.


## Update the manifest.json

The `manifest.json` will need to be updated to include the `identity` permission to 
use the `chrome.identity` methods.

If you're using `getAuthToken` then you'll also need to update the [`oauth2`](https://developer.chrome.com/docs/extensions/how-to/integrate/oauth) section to include the scopes and client ID. Note, that the `oauth2` section is only used with `getAuthToken` and not with `launchWebAuthFlow`.

```json
"oauth2": {
  "client_id": "<client-id>",
  "scopes": [
    "https://www.googleapis.com/auth/userinfo.email",
    "https://www.googleapis.com/auth/userinfo.profile"
  ]
}
```

## Update the authorized callback URL in the Google Cloud Console

The authorized callback URL in the Google Cloud Console needs to match the redirect URL that you are using in the extension. This is important because the OAuth flow will redirect back to the extension with the authorization code. If the redirect URL does not match the authorized callback URL in the Google Cloud Console then the OAuth flow will fail.

So, in your Googlle Cloud project, navigate to:

**Credentials > OAuth 2.0 Client IDs > Web client (auto created by Google Service) > Edit**

And, add the value that you have for `redirectUri` to the authorized callback URLs for the OAuth client:

```
https://${chrome.runtime.id}.chromiumapp.org
```

## Next steps

This is the first step in the OAuth flow. The next post covers the [authorization code exchange for the user access token and refresh token](/post/chrome-extension-google-oauth-access-token/). This will require a server process to handle the token exchange and refresh. We'll use a Cloudflare Worker to handle this process.

If this post was helpful, consider signing up for the newsletter (below) to get updates when new posts are published.