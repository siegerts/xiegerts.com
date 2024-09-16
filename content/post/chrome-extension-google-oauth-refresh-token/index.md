+++
title = "Handling Google OAuth Refresh Tokens in a Chrome Extension"
description = "How to handle the token refresh for the Google OAuth flow in a Chrome extension using an external server API running in a Cloudflare Worker."

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
    "refresh token",
    "hono",
    "cloudflare worker",
    "API"
]

categories = ["Development"]
date = 2024-09-14T15:25:16-04:00

series = "Developing Browser Extensions"

+++

This is the last post in the series on developing browser extensions with Google Sign In using Google OAuth. In the previous post, we looked at how to [exchange the Google OAuth authorization code for the access token and refresh token in a Cloudflare Worker API](/post/chrome-extension-google-oauth-access-token/). In this post, we will look at how to handle the token refresh in the Chrome extension using the refresh token that we got from the token exchange API.

Now that we have the access token and refresh token, we can use the refresh token to get a new access token when the current access token expires. This is important because the access token that we get from Google will expire after an hour, and we don't want user to have to sign in again each time the token expires.

There are are few updates to the existing code that we need to make to handle the token refresh:

1. Update the API with a new endpoint to handle the token refresh
2. Update the Chrome extension to use the refresh token to get a new access token
3. Protect the refresh token endpoint with Firebase Auth (optional)



## Update the API with a new endpoint to handle the token refresh


```ts
//...

app.post("/api/auth/refresh", async (c) => {
	const clientId = c.env.CLIENT_ID;
	const clientSecret = c.env.CLIENT_SECRET;

	const body = await c.req.json();
	const now = Math.floor(Date.now() / 1000);

	try {
		const response = await fetch("https://oauth2.googleapis.com/token", {
			method: "POST",
			headers: {
				"Content-Type": "application/json",
			},
			body: JSON.stringify({
				refresh_token: body.refresh_token,
				client_id: clientId,
				client_secret: clientSecret,
				grant_type: "refresh_token",
			}),
		});

		if (response.ok) {
			const { access_token: accessToken, expires_in: expiresIn } =
				(await response.json()) as OAuthTokenResponse;

			return c.json(
				{
					accessToken,
					expiresAt: now + expiresIn,
				},
				200,
			);
		}

		// handle for invalid refresh token
		const data = await response.json();
		console.error("request failed: ", data);
		return c.json(data, 400);
	} catch (error) {
		console.error("Error with refresh_token request: ", error);
		return c.json({ error }, 400);
	}
});

//...
```

This code is similar to the code we used to exchange the authorization code for the access token and refresh token, but instead of using the `authorization_code` grant type, we are using the `refresh_token` grant type to get a new access token using the refresh token. Also, the redirect URI is not required when using the refresh token grant type.

The environment variables `CLIENT_ID` and `CLIENT_SECRET` are required again to make the request to the Google OAuth API. The `refresh_token` is sent in the request body to get a new access token.


## Update the Chrome extension to use the refresh token to get a new access token

So, now that we have the refresh token, we can use it to get a new access token when the current access token expires. We can create a utility function to handle the token refresh in the Chrome extension.

This is just an example of how to handle the refresh and persist the new access token in the Chrome  local storage. This is not a definitive guide - make sure to update as needed.

```ts

async function getToken() {
  chrome.storage.local.get(["accessToken", "refreshToken", "expiresAt"], async (items) => {
    const { accessToken, refreshToken, expiresAt } = items;

    if (accessToken) {
      const nowInSeconds = Math.floor(Date.now() / 1000);
      const nowPlus60 = nowInSeconds + 60;

      // expired or will expire in the next 60 seconds
      if (expiresAt <= nowPlus60) {
        const response = await fetch(`${BASE_API_URL}/api/auth/refresh`, {
          method: "POST",
          headers: {
            "Content-Type": "application/json",
          },
          body: JSON.stringify({
            refresh_token: refreshToken,
          }),
        });

        if (response.ok) {
          const { accessToken, expiresAt } = await response.json();
          chrome.storage.local.set({ accessToken, expiresAt });

          console.log("Access token refreshed");
          return accessToken;

        } else {
          const data = await response.json();
          console.error("request failed: ", data);
        }
      } else {
        console.log("Access token is still valid");
        return accessToken;
      }
    
    }
  });

}
```

This function checks if the access token is expired or will expire in the next 60 seconds. If it is, it makes a request to the `/api/auth/refresh` endpoint to get a new access token using the refresh token. If the request is successful, it updates the `accessToken` and `expiresAt` in the Chrome storage. If the access token is still valid, it returns the current access token.

This function can be called before making any API requests to ensure that the access token is always valid.

The 60 seconds buffer is added to account for clock skew between the client and the server that has been introduced during the token exchange process.

In production, you'll definitely want to add more error handling, logging, and probably abstract this into a separate functions for better reuse. And, if you're using Firebase Auth, you'll need to account for handling the signed in Firebase user if the access token expires or is revoked.


## Protect the refresh token endpoint with Firebase Auth (optional)

If you're using Firebase Auth to manage user authentication, you can protect the refresh token endpoint with Firebase Auth to ensure that only authenticated users can access the endpoint to refresh their access token.

Using [Firebase Auth middleware](https://github.com/honojs/middleware/tree/main/packages/firebase-auth) with Hono, we can protect the endpoint:

```ts
//...

app.use("/api/auth/refresh", async (c, next) => {
	const auth = verifyFirebaseAuth({
		projectId: c.env.FIREBASE_PROJECT_ID,
	} as VerifyFirebaseAuthConfig);

	return auth(c, next);
});

app.post("/api/auth/refresh", async (c) => {
  const tok = getFirebaseToken(c);
  const uid = tok?.uid;

  if (!uid) {
    return c.json({ error: "Unauthorized" }, 401);
  }

  // ...
});

```

This will require some additional configuration in the Cloudflare Worker environment.


## Revoking tokens on user sign out 

If you want to revoke the access token on user sign out you can do that from the extension by sending a request to the `/api/auth/revoke` endpoint with the access token.

```ts
await fetch(
  `https://accounts.google.com/o/oauth2/revoke?token=${token}`,
)
```


## Handling revoked tokens

There are a few scenarios where the [access token can be revoked](https://developers.google.com/identity/protocols/oauth2#expiration) that you should think about when implementing token refresh. A few are:

1. User signs out of the extension
2. User revokes the app permissions from their Google account
3. Access token expires
4. Access token is revoked by Google

There are also API limits on a number of refresh tokens that can be issued for a user account. 

In each of these cases, you should handle the token revoke by clearing the access token and refresh token from the Chrome storage and updating the UI to reflect the user's sign-in state. Then you can prompt the user to sign in again if needed.


# Conclusion

Hopefully these posts help unblock you if you're working on a similar project :coffee: :tea:

If this post was helpful, consider signing up for the newsletter (below) to get updates when new posts are published.