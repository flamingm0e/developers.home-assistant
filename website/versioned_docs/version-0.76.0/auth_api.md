---
title: Authentication API
sidebar_label: API
id: version-0.76.0-auth_api
original_id: auth_api
---

This page will describe the steps required for your application to authorize against and integrate with Home Assistant instances.

Each user has their own instance of Home Assistant which gives each user control over their own data. However, we also wanted to make it easy for third party developers to create applications that allow users to integrate with Home Assistant. To achieve this, we have adopted the [OAuth 2 specification][oauth2-spec] combined with the [OAuth 2 IndieAuth extension][indieauth-spec] for generating clients.

## Clients

Before you can ask the user to authorize their instance with your application, you will need a client. In traditional OAuth2, the server needs to generate a client before a user can authorize. However, as each server belongs to a user, we've adopted a slightly different approach from [IndieAuth][indieauth-clients].

The client ID you need to use is the website of your application. The redirect url has to be of the same host and port as the client ID. For example:

 - client id: `https://www.my-application.io`
 - redirect uri: `https://www.my-application.io/hass/auth_callback`

If you require a different redirect url (ie, if building a native app), you can add an HTML tag to the content of the website of your application (the client ID) with an approved redirect url. For example:

```html
<link rel='redirect_uri' href='hass://auth'>
```

Home Assistant will scan the first 10kB of a website for these tags.

## Authorize

[![Authorization flow sequence diagram](/img/en/auth/authorize_flow.png)](https://www.websequencediagrams.com/?lz=dGl0bGUgQXV0aG9yaXphdGlvbiBGbG93CgpVc2VyIC0-IENsaWVudDogTG9nIGludG8gSG9tZSBBc3Npc3RhbnQKABoGIC0-IFVzZXI6AEMJZSB1cmwgAD4JACgOOiBHbyB0bwAeBWFuZCBhAC0ICgBQDgB1DACBFw5jb2RlAHELAE4RZXQgdG9rZW5zIGZvcgAoBgBBGlQAJQUK&s=qsd)

> All example URLs here are shown with extra spaces and new lines for clarity. Those should not be in the final url.

The authorize url should contain `client_id` and `redirect_uri` as query parameters.

```
http://your-instance.com/auth/authorize?
    client_id=ABCDE&
    redirect_uri=https%3A%2F%2Fexample.com%2Fhass_callback
```

Optionally you can also include a `state` parameter, this will be added to the redirect uri. The state is perfect to store the instance url that you are authenticating with. Example:

```
http://your-instance.com/auth/authorize?
    client_id=ABCDE&
    redirect_uri=https%3A%2F%2Fexample.com%2Fhass_callback&
    state=http%3A%2F%2Fhassio.local%3A8123
```

The user will navigate to this link and be presented with instructions to log in and authorize your application. Once authorized, the user will be redirected back to the passed in redirect uri with the authorization code and state as part of the query parameters. Example:

```
https://example.com/hass_callback?
    code=12345&
    state=http%3A%2F%2Fhassio.local%3A8123
```

This authorization code can be exchanged for tokens by sending it to the token endpoint (see next section).

## Token

The token endpoint returns tokens given valid grants. This grant is either an authorization code retrieved from the authorize endpoint or a refresh token.

All interactions with this endpoint need to be HTTP POST requests to `http://your-instance.com/auth/token` with the request body encoded in `application/x-www-form-urlencoded`.

### Authorization code

> All requests to the token endpoint need to contain the exact same client ID as was used to redirect the user to the authorize endpoint.

Use the grant type `authorization_code` to retrieve the tokens after a user has successfully finished the authorize step. The request body is:

```
grant_type=authorization_code&
code=12345&
client_id=https%3A%2F%2Fwww.home-assistant.io%2Fios
```

The return response will be an access and refresh token:

```json
{
    "access_token": "ABCDEFGH",
    "expires_in": 1800,
    "refresh_token": "IJKLMNOPQRST",
    "token_type": "Bearer"
}
```

The access token is a short lived token that can be used to access the API. The refresh token can be used to fetch new access tokens. The `expires_in` value is seconds that the access token is valid.

### Refresh token

Once you have retrieved a refresh token via the grant type `authorization_code`, you can use it to fetch new access tokens. The request body is:

```
grant_type=refresh_token&
refresh_token=IJKLMNOPQRST
```

The return response will be an access token:

```json
{
    "access_token": "ABCDEFGH",
    "expires_in": 1800,
    "token_type": "Bearer"
}
```

## Making authenticated requests

Once you have an access token, you can make authenticated requests to the Home Assistant APIs.

For the websocket connection, pass the access token in the [authentication message](https://developers.home-assistant.io/docs/en/external_api_websocket.html#authentication-phase).

For HTTP requests, pass the token type and access token as the authorization header:

```
Authorization: Bearer ABCDEFGH
```

If the access token is no longer valid, you will get a response with HTTP status code 401 unauthorized. This means that you will need to refresh the token. If the refresh token doesn't work, the tokens are no longer valid and so the user is no longer logged in. You should clear the user's data and ask the user to authorize again.

[oauth2-spec]: https://tools.ietf.org/html/rfc6749
[indieauth-spec]: https://indieauth.spec.indieweb.org/
[indieauth-clients]: https://indieauth.spec.indieweb.org/#client-identifier
