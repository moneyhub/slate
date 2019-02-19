# Moneyhub Identity Service

###### Version 0.8

We provide an OpenID Connect compliant interface that should work well with any OpenID Connect certified relying party software.

This document will provide a high level overview, but we recommend that users familiarise themselves with the following specs:

- [OpenID Connect Core](http://openid.net/specs/openid-connect-core-1_0.html)
- [Financial Grade API Read Only Profile](https://bitbucket.org/openid/fapi/src/master/Financial_API_WD_001.md)
- [Financial Grade API Read/Write Profile](https://bitbucket.org/openid/fapi/src/master/Financial_API_WD_002.md)

## Overview

Our identity service supports the following use cases:

1. Allowing a user to connect to a financial institution and grant permissioned access to their data from that financial institution.
2. Allowing a user to connect to multiple financial institutions through a single profile and gain access to the data from those institutions.

We provide these features via an OpenID Provider interface that supports standard OAuth 2 based flows to issue access tokens that can be used to gain access to financial data via our API Gateway.

For API documentation please see [here](https://api.moneyhub.co.uk/docs).

### Flow for use case 1

- Partner redirects user to identity service `/oidc/auth` with a scope param that contains the id of the bank to connect to and the level of data to gain consent for
- Moneyhub Identity service gains consent from the user to access their banking data
- Moneyhub Identity service redirects the user to the bank
- Bank authenticates the user and sends them back to the Moneyhub Identity Service
- Moneyhub redirects the user back to the partner with an `authorization_code`
- Partner exchanges this code for an `access_token`
- Partner uses the access token at the api gateway to access financial data

### Flow for use case 2

- Partner requests an access token from the identity service with the scope `user:create`
- Partner uses this token to create a profile at the /user endpoint
- Partner redirects user to the identity service `/oidc/auth` with a scope param that contains the id of the bank to connect to, and with the id of the new user profile in the claims parameter
- Moneyhub Identity service gains consent from the user to access their banking data
- Moneyhub Identity service redirects the user to the bank
- Bank authenticates the user and sends them back to the Moneyhub Identity Service
- Moneyhub redirects the user back to the partner with an `id_token` that contains the `connection_id`
- Partner requests an access token from the identity service with the scope of data access required and a custom `sub` parameter that contains the profile id
- Partner uses the access token at the api gateway to access financial data

More detailed examples of the above flows are available later in this document.

## Scopes

We use scopes to both describe the access the user is granting and the way in which you would like the user to identify themselves.

Below is a summary of the scopes we provide, please check our discovery document (available at /oidc/.well-known/openid-configuation to see which particular scopes are supported by a given deployment of our identity service).

### Scopes indicating the manner in which we should authenticate the user:

- `id:{bank_code}` - if you pass a specific bank code (available via the endpoints listed above) then we will bypass the bank chooser and take the user directly to the selected bank.
- `id:all` - if you specify this scope we will display a list of all available connections
- `id:api` - if you specify this scope we will display a list of the available API based connections
- `id:legacy` - if you specify this scope we will display a list of the available legacy (screen scraping) connections
- `id:test` - if you specify a type of connection then we will display a list of our test connections. This scope will be enabled by default when creating a new client

The above scopes are mutually exclusive and we will return
an error of `invalid_scope` if more than one of the above is supplied.

### Scopes describing data to be requested:

- `transactions:read:all` - All transactions
- `transactions:read:in` - All incoming transactions
- `transactions:read:out` - All outgoing transactions

Note - the above transactions:read scopes are mutually exclusive - if more than one is provided there will be an `invalid_scope` error.

- `transactions:write` - For all transactions that are able to be read it is possible to edit certain fields (e.g. category, notes, etc.). Please see the documentation on the transactions endpoint for details of which fields can be edited. If `transactions:write` is provided without any `transactions:read` scope there will be an `invalid_scope` error.
- `transactions:write:all` - This allows full access to create transactions, edit all their properties and delete transactions. This scope is only available when issuing tokens for users that are managed by the client - i.e. use case 2.
- `accounts:read` - Read access to all accounts
- `accounts:write` - Write access to all accounts. Please see the accounts endpoint for details of which fields can be edited.
- `accounts:write:all` - Full write access including the ability to delete accounts. This scope is only available when issuing tokens for users that are managed by the client - i.e. use case 2.
- `categories:read` - Read access to a customer's categories.
- `categories:write` - Write access to a customer's custom categories.
- `spending_analysis:read` - Read access to a customer's spending analysis.
- `spending_goals:read` - Read access to a customer's spending goals.
- `spending_goals:write` - Write access to a customer's spending goals.
- `spending_goals:write:all` - Full write access to spending goals, including the ability to delete goals.
- `savings_goals:read` - Read access to a customer's saving goals.
- `savings_goals:write` - Write access to a customer's saving goals.
- `savings_goals:write:all` - Full write access to saving goals, including the ability to delete goals.

### Scopes describing connection lifecycle flows:

Some of the OpenBanking APIs that we connect to require the user to re-authenticate every 90 days. In addition we have screen-scraping connections that the user will need to update if their credentials change. In order to support these flows we support the following scopes:

- `reauth`
- `refresh`

These scopes require a claims parameter to be sent that contains a `sub` value and a `mh:con_id` value. Moneyhub will then take the user through a re-authentication journey or "refresh" journey.

We advise that the above 2 scopes are used with the response_type of `id_token` rather than `code id_token`. This is because the access token issued at the end of such a flow is of no value and can only be introspected to confirm the values already present in the id_token.

The only scope that can (and must) be supplied along with either `reauth` or `refresh` is `openid`. If any other scope is provided the result will be an `invalid_scope` error.

### Other Scopes

- `openid` - this scope is required and indicates that you are using our OpenID Connect interface. We will return an id token as described in the OpenID Connect Core spec. You can request specific claims to be present in the id token using the claims parameter described in OpenID Connect Core. For details on what claims we support, please see below.
- `offline_access` - this scope indicates that you would like ongoing access to the user's resources, when it is present we will issue a refresh token
- `user:create` - this scope is only supported with the client credentials grant type. It allows a relying party to create a new user profile.
- `user:read` - this scope is only supported with the client credentials grant type. It allows a relying party to access their user profiles.

## Claims

> To add a new account for a registered user via either openbanking or
screen scraping the following parameters would be sent in the request object:

```javascript
{
  "scope": "openid id:all",
  "claims": {
    "id_token": {
      sub: {
        essential: true,
        value: "24400320",
      },
      "mh:con_id": {
        "essential": true
      }
    }
  }
}
```

> On completion of the connection, the connection id is returned in the id token as below:

```json
{
  "iss": "https://identity.moneyhub.com",
  "sub": "24400320",
  "aud": "s6BhdRkqt3",
  "nonce": "n-0S6_WzA2Mj",
  "exp": 1311281970,
  "iat": 1311280970,
  "mh:con_id": "the-connection-id"
}
```

Moneyhub uses the OpenID Connect [claims parameter](http://openid.net/specs/openid-connect-core-1_0.html#ClaimsParameter) for the following purposes:

1. Specifying the connection that should be re-authorised or refreshed
2. Specifying the user profile that an account should be added to

The format of the claims parameter may seem odd to those unfamiliar with OpenID Connect, but it has the advantage of being a standards compliant technique of supporting the above purposes. It is supported by many OpenID Connect relying party libraries.

Our discovery document details the `claims` that we support, they currently include:

- `sub` - the subject (user id) that the authorization request should be scoped to (for adding, reauth and refresh)
- `mh:con_id` - the connection id that the authorization request should be scoped to (for reauth and refresh)

## Our OpenID Connect Implementation

Moneyhub supports the following endpoints:

### Authorization Endpoint

As described [here](http://openid.net/specs/openid-connect-core-1_0.html#AuthorizationEndpoint)

We support the use of request objects and the claims parameter at this endpoint.

### Token Endpoint

As described [here](http://openid.net/specs/openid-connect-core-1_0.html#TokenEndpoint)

We support the following grant types:

- `authorization_code`
- `client_credentials`
- `refresh_token`

The `authorization_code` and `refresh_token` grant types are implemented exactly as according to the specs in [RFC6749](https://tools.ietf.org/html/rfc6749) and [OIDC](http://openid.net/specs/openid-connect-core-1_0.html).

The `client_credentials` grant supports a custom parameter of `sub`. This is a required parameter for any scope apart from the `user:create` or `user:read` scopes.

### Discovery

As described [here](http://openid.net/specs/openid-connect-discovery-1_0.html)

Our discovery document is available [here](https://identity.moneyhub.co.uk/oidc/.well-known/openid-configuration).
It will contain our up-to-date machine readable configuration
and for example will list our:

- token endpoint
- authorization endpoint
- jwks endpoint
- scopes that we support
- claims that we support
- the cryptographic algorithms we support
- the response types we support

Examples of discovery metadata from other providers are:

- [Google](https://accounts.google.com/.well-known/openid-configuration)
- [Microsoft](https://login.windows.net/contoso.onmicrosoft.com/.well-known/openid-configuration)

### Bank connections

We have 4 lists of available bank connections:

- [all connections](https://identity.moneyhub.co.uk/oidc/.well-known/all-connections)
- [API connections](https://identity.moneyhub.co.uk/oidc/.well-known/api-connections)
- [screen-scraping connections](https://identity.moneyhub.co.uk/oidc/.well-known/legacy-connections)
- [test connections](https://identity.moneyhub.co.uk/oidc/.well-known/test-connections)

Every client you create will have access to the test connections by default. Access to the real connections via the API
will need to be requested.

Every connection will have the following properties:
- `id` - bank connection id (used to request an authorization url for a specific bank)
- `name`
- `type` - the type of bank connection (`api`, `legacy` or `test`)
- `bankRef` - reference that uniquely identities a set of connections as being part of the same institution (e.g. HSBC Open banking and HSBC credit cards). It is used to group a set of connections by the banking institution they refer to
- `parentRef` - this property is now deprecated. Please use `bankRef` instead
- `iconUrl` - the url of the bank icon SVG. Please be aware we don't have icons for all the connections we provide. For the missing icons you can either use your own set or use our generic bank icon found at this url: `https://identity.moneyhub.co.uk/bank-icons/default`
- `accountTypes` - an array containing the types of accounts supported by the connection (`cash`, `card`, `pension`, `mortgage`, `investment`, `loan`) and a beta boolean value flagging which accounts types for that connection are currently being developed and may not have a 100% success rate
- `userTypes` - an array of user account types supported by the bank connection (`personal` and `business`)

### Response Types

Our discovery doc will list the response types that we support. Currently these are: `code`, `code id_token` and `id_token`.

`code` is fairly straight forward and is the standard OAuth 2
authorization code flow.

`code id_token` is one of the variants of the hybrid flow and isn't always understood. At a basic level it means that we will send an `id_token` along with the authorization code when we redirect the user back to your `redirect_uri`. This id_token doesn't contain any identity information, but is rather a detached signature which cryptographically binds the authorization_code we send back with the nonce and the state that you sent to us. It prevents a certain class of code interception attacks and we encourage implementers to use it rather than the basic authorization code flow.

### Request Object

OpenID Connect defines the request object - this is a JWT that contains the standard OAuth 2.0 parameters such as: `client_id`, `scope`, `state`, etc.

We encourage implementers to use this and may require it for certain use cases.

The request object allows you to sign the request parameters and prevents tampering. This can prevent a certain class of attacks against OAuth 2.0.

More information about request objects is available here:
http://openid.net/specs/openid-connect-core-1_0.html#JWTRequests

### JWKS Endpoints & Asymmetric Signatures

We use jwks endpoints to support the rotation of signing keys:
http://openid.net/specs/openid-connect-core-1_0.html#RotateSigKeys

As part of registering your client software with us, we will ask you for your own jwks endpoint.

If you don't yet have one, we strongly encourage you to implement one. JWKS endpoints provide a neat method to achieve key rotation without any interruption to service or any need for bilateral communication.

They enable you to manage your keys in which ever manner you see fit and removes the need for the `client_secret`.

For more information about the benefits of this approach or advice
on implementing, please contact us.

## Registration

We plan to support dyanamic client registration (https://openid.net/specs/openid-connect-registration-1_0.html), but currently in order to register your software with us we will ask you to send the following information:

- `redirect_uris`
- `client_name`
- `logo_uri`
- `jwks_uri`
- `id_token_signed_response_alg`
- `request_object_signing_alg`

Definitions of the above can be found here:
https://openid.net/specs/openid-connect-registration-1_0.html#ClientMetadata

## User Management

To support use case 2, the following RESTful routes are available:

### POST /users

> Example request:

```
POST /users HTTP/1.1
Host: identity.moneyhub.co.uk
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6IkQ5SkFObVlmU0dfa2J0MktScFRLbzdRQ05IMl9SSy0wYTc4N3lqbTA3encifQ.eyJqdGkiOiJjcEVPMk11dVJscmtDfkdDQ3Rqa0IiLCJpc3MiOiJodHRwOi8vaWRlbnRpdHkuZGV2LjEyNy4wLjAuMS5uaXAuaW8vb2lkYyIsImlhdCI6MTUzNDQ5MzU0MSwiZXhwIjoxNTM0NDk0MTQxLCJzY29wZSI6InVzZXI6Y3JlYXRlIiwiYXVkIjoiODk4YzUyOWItYzA2Mi00ZjI2LWExMzYtZmQ4YmM0NjJkNTgzIn0.AMU266O-wgmz-6SOfSF_Bq0LQhoAgytaInwCKhT-tXQ6Z_L0I75blmRujnKALK-LG08ny_gemtDWUEmD2mjyHgO-vtmiSNMHF2T5z2GS3k4VOUbGKVjFY5kK9QfoUCR_WCpUEPd64LHe_IaR0rMAzaKcVLRhtjin9yAB-goif683ESBFQLDrnojzdcOxWtP1x_qGSNBOMqJ6RDk7H65aBCXJj5eee11EW71G1Q3C3_MyJqTYdwXbAzkE-8XLDznDqZzVmm4erFUTN3TuB5L7af2pendAWitGEeshHKRpgeHI3EQrNj98-UIyemVV9tUK76x2ojiV1ge7ZpnYeNCO0A
Content-Type: application/json

{
  "clientUserId":"some-id"
}
```

> Example response:

```json
{
  "clientUserId": "some-id",
  "userId": "328278302947678c0fc37f54",
  "clientId": "898c529b-c062-4f26-a136-fd8bc462d583",
  "scopes": "user:create",
  "managedBy": "client"
}
```

This allows an API client to create a new "user". Once this user has been created the following operations are possible:

- starting an authorisation flow to connect a financial provider to that user
- gaining an access token for that user
- using that access token to get and create financial resources for that user

This route requires an access token from the client credentials grant with the scope of `user:create`.

It accepts a JSON body with a single parameter: `clientUserId`. This is optional but allows an API cilent to persist it's own identifier against the user.

### GET /users

This route requires an access token from the client credentials grant with the scope of `user:read`.
It returns an array of all the users associated with your api client.

### GET /users/:id

This route requires an access token from the client credentials grant with the scope of `user:read`.
It returns a single user associated with your api client.

## Example for use case 2

```javascript
const { access_token } = await client.grant({
  grant_type: "client_credentials",
  scope: "user:create",
})

const user = await got.post(`#{identityServiceUrl}/`, {
  headers: {
    Authorization: `Bearer ${access_token}`,
  },
  json: true,
  body: { clientUserId: "some-id" },
})

const authParams = {
  client_id: client_id,
  scope: "openid id:some-bankid",
  state: "your-state-value",
  redirect_uri: redirect_uri,
  response_type: "code",
  prompt: "consent",
}

const claims = {
  id_token: {
    sub: {
      essential: true,
      value: user.userId,
    },
    mh:con_id: {
      essential: true,
    },
  },
}

const request = await client.requestObject({
  ...authParams,
  claims,
  max_age: 86400,
})

const url = client.authorizationUrl({
  ...authParams,
  request,
})

// redirect the user to this url
// if everything is succesful they will be redirected to your `redirect_uri` with `code` and `state` as query parameters.

const tokens = await client.authorizationCallback(
  redirect_uri,
  { code, state },
  { state: expectedState }
)
// these tokens confirm that the account has been added, however cannot be used for data access
// the `id_token` will contain the id of the connection that has just been created

const { access_token } = await client.grant({
  grant_type: "client_credentials",
  scope: "accounts:read transactions:read:all",
  sub: user.userId,
})

const accounts = await got(`#{resourceServerUrl}/accounts`, {
  headers: {
    Authorization: `Bearer ${access_token}`,
  },
  json: true,
})

const transactions = await got(`#{resourceServerUrl}/transactions`, {
  headers: {
    Authorization: `Bearer ${access_token}`,
  },
  json: true,
})
```

This example assumes the use of an OpenID Client (e.g. https://github.com/panva/node-openid-client)