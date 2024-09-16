+++
title = "Handling Google OAuth Authorization Code and Access Token Exchange in a Chrome Extension"

description = "How to handle the token exchange and refresh for the Google OAuth flow in a Chrome extension using an external server API running in a Cloudflare Worker and Firebase Auth."

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
    "authorization",
    "hono",
    "cloudflare worker",
    "API",
    "access token",
]

categories = ["Development"]
date = 2024-09-14T15:25:16-04:00

series = "Developing Browser Extensions"

+++


This is a continuation of the series on developing browser extensions with Google Sign In using Google OAuth. In the previous post, we looked at how to [set up a Chrome extension with `chrome.identity.launchWebAuthFlow` and Google Sign In](/post/chrome-extension-oauth-web-auth-flow-firebase-google/) as an alternative to `chrome.identity.getAuthToken`. In this post, we will look at how to handle the token exchange and refresh for the Google OAuth flow in a Chrome extension using and external server API running in a Cloudflare Worker.


There are two case that we need to handle with the API:
1. The initial token exchange to get the access token and refresh token from the authorization code returned by the OAuth flow
2. The token refresh to get a new access token when the current access token expires


## Setting up a Cloudflare Worker to handle the token exchange

This API uses the [Hono framework](https://hono.dev/docs/getting-started/cloudflare-workers) and handles the token exchange after the OAuth flow has completed. 

{{< note >}}

I use Cloudflare Workers for this example but you can use any platform like AWS Lambda, Google Cloud Functions, or persistent serer will work.

{{< /note >}}


The extension will:
1. Get the authorization code from the OAuth flow
2. Send the authorization code to the Cloudflare Worker API
3. The API will exchange the authorization code for the access token and refresh token
4. The API will return the access token and refresh token to the extension


```ts
import { Hono } from "hono";

type Env = {
	CLIENT_ID: string;
	CLIENT_SECRET: string;
	REDIRECT_URI: string;
};

type OAuthTokenResponse = {
	access_token: string;
	expires_in: number;
	id_token: string;
	scope: string;
	token_type: string;
	refresh_token?: string;
};

const app = new Hono<{ Bindings: Env }>();


app.post("/api/auth/token", async (c) => {
	const clientId = c.env.CLIENT_ID;
	const clientSecret = c.env.CLIENT_SECRET;
	const redirectUri = c.env.REDIRECT_URI;

	const body = await c.req.json();
	const now = Math.floor(Date.now() / 1000);

	try {
		const response = await fetch("https://oauth2.googleapis.com/token", {
			method: "POST",
			headers: {
				"Content-Type": "application/json",
			},
			body: JSON.stringify({
				code: body.code,
				client_id: clientId,
				client_secret: clientSecret,
				redirect_uri: redirectUri,
				grant_type: "authorization_code",
			}),
		});

		if (response.ok) {
			const {
				access_token: accessToken,
				expires_in: expiresIn,
				refresh_token: refreshToken,
			} = (await response.json()) as OAuthTokenResponse;

			return c.json(
				{
					accessToken,
					expiresAt: now + expiresIn,
					refreshToken,
				},
				200,
			);
		}

		const data = await response.json();
		console.error("request failed: ", data);
		return c.json(data, 400);
	} catch (error) {
		console.error("Error with authorization_code request: ", error);
		return c.json({ error }, 400);
	}
});


export default app;

```

Stepping through the code:
1. The required ENV variables are the `CLIENT_ID`, `CLIENT_SECRET`, and `REDIRECT_URI` which are used to make the request to the Google OAuth API. These are set in the Cloudflare Worker environment variables (`wrangler.toml`) and orginiate from the Google Cloud Console OAuth client settings. The `REDIRECT_URI` must match the one that was set in the initial OAuth flow request from the extension.
2. The API listens for POST requests to `/api/auth/token` from the extension and expects a JSON body with the `code` from the OAuth flow.
3. A request is made to the Google OAuth API to exchange the `code` for the `access_token` and `refresh_token`.
4. If the request is successful, the `access_token`, `expiresAt`, and `refresh_token` are returned to the extension.
5. `expiresAt` is calculated by adding the `expires_in` value to the current time in seconds. This is used by the extension to determine when the access token will expire. Access tokens from Google expire after an hour but this will account for any discrepancies in expiration time.

The `grant_type` is set to `authorization_code` since the API is exchanging the authorization code for the access token and refresh token.

{{< note >}}

Note, the `"Content-Type": "application/json"` header allows the payload to be sent as JSON. If `"Content-Type": "application/x-www-form-urlencoded"` is used, the body should be formatted as a query string.

{{< /note >}}

After the API is set up and deployed, the extension can now make requests to the API to exchange the authorization code for the access token and refresh token.

## Update the extension `manifest.json` host permissions

The `host_permissions` in the `manifest.json` will need updated to since the extension will need to make requests to the external API:

```json
"host_permissions": [
  "<api-endpoint>/*"
],
```


## Updating the Chrome Extension to handle the token exchange

Now, back in the extension, we need to update the callback function from the OAuth flow to send the authorization code to the API. The extension will then store the access token and refresh token in the Chrome storage.

```diff
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
+         let response: Response
+         try {
+           response = await fetch(
+             `${API_BASE_URL}/api/auth/token`,
+             {
+               method: "POST",
+               headers: {
+                 "Content-Type": "application/json",
+               },
+               body: JSON.stringify({
+                 code,
+               }),
+             },
+           )
+
+           const { accessToken, expiresAt, refreshToken } =
+             await response.json()
+
+           if (accessToken) {
+             // save the tokens and expiration time to Chrome Storage
+             await chrome.storage.local.set({
+               accessToken,
+               refreshToken,
+               expiresAt,
+             })
+           }
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

If you're making the request for the access and refresh token from the server but the `refresh_token` is not included in the response, then you'll need to reauthorize the user to get the `refresh_token` again. This is because the `refresh_token` is **only returned the first time the user authorizes the app**.

The app permissions can be revoked at https://myaccount.google.com/permissions. So if you've been testing along the way, you'll likely need to revoke the permissions to get the `refresh_token` again.



## Signing user in with Firebase using the access token (optional)

Now that the extension has the access token and refresh token, it can use the access token to sign the user in with Firebase. This is done by creating a `GoogleAuthProvider` credential with the access token and then signing in with the credential.

```ts
//...

const credential = GoogleAuthProvider.credential(
  null,
  accessToken,
)

              
await signInWithCredential(getAuth(), credential)

//...
```


Now, the extension use [`onAuthStateChanged`](https://firebase.google.com/docs/reference/js/v8/firebase.auth.Auth#onauthstatechanged) to listen for changes in the user's sign-in state and update the UI accordingly.

```ts
onAuthStateChanged(auth, (user) => {
  //...
})

```

Nice, now the extension can handle the token exchange for the Google OAuth flow and sign the user in with Firebase (optional). This allows the extension to access Google APIs and Firebase services with the user's credentials.

The next post will look at how to [handle the OAuth token refresh when the access token expires](/post/chrome-extension-google-oauth-refresh-token/).

If this post was helpful, consider signing up for the newsletter (below) to get updates when new posts are published.


