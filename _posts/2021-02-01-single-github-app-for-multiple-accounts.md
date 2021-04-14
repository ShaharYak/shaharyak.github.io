---
layout: post
title: Using a single GitHub App when working with multiple accounts
description: How to use GitHub API to manage access tokens and installation IDs to enable working with multiple GitHub accounts within your application
date: 2021-02-01
img: github-app.jpg
tags: [GitHub, GitHub App]
---

## How to use GitHub API to manage access tokens and installation IDs to enable working with multiple GitHub accounts within your application

<p align="center">
  <img src="https://miro.medium.com/max/700/1*ceTdFZnQqpLD3CNp7I0NTA.png">
</p>

**Originally published on [Altostra](https://miro.medium.com/max/700/1*pP2DLP90JS1UaQ4DWl-j7Q.jpeg)**


Altostra is a no-code infrastructure platform for developers and teams to accelerate cloud applications delivery.

As a developer-first platform, we use git providers for identity as well as for source control. In this article, we’ll discuss how we utilize GitHub for managing multiple teams on Altostra.

Using the [Altostra GitHub App](https://github.com/marketplace/altostra), we can integrate with our users’ GitHub accounts and perform operations, such as creating a repository or committing files, as Altostra. This enables our users to manage their Altostra projects inside their GitHub accounts.

## Storytime

Bob is a developer in an organization with multiple development teams that use Altostra.

First, Bob connects his GitHub account by installing the Altostra GitHub App. After that,  Bob can immediately start working with Altostra by importing existing projects from GitHub and creating new Altostra projects from templates.

Later, Bob creates a new team and wants to connect his GitHub account to it. This time, we can’t use the previous GitHub installation workflow because the app was already installed on Bob’s GitHub account. So, we [list](https://docs.github.com/en/rest/reference/apps#list-app-installations-accessible-to-the-user-access-token) all the GitHub accounts that Bob has access to and already have the Altostra GitHub App installed.

Now, Bob can choose an existing installation or connect a new one.

![](../images/blog/single-github-app-for-multiple-accounts/github-choose-teams.png)

## The Challenge

At first, the relation between an Altostra account to a GitHub App installation (AKA **“installation”**) was a 1–1 relation, which means that a GitHub account could be connected to only a single Altostra account.

This limitation created a challenge because our users need to use the same GitHub App installation across many accounts, which breaks the 1–1 relation.

To overcome this challenge, we had to come up with a way to manage the GitHub App installations with Altostra’s accounts and allow the reuse of the installations on many accounts.

In Bob’s story, we used two types of access tokens, a “user access token” and an “app access token” each is used for different purposes. Let’s dive in.

## GitHub access tokens

While some Github API operations don’t require any special permissions (listing pubic repositories, getting public user data, and so on), others require authentication in the form of an access token, such as create a repository, create PR, commit files, etc.

We use two types of tokens:

* User access tokens — Allows us to act on the user’s behalf.

* App access tokens — Allows us to act on the app’s behalf.

![Installed GitHub Apps = App access token. Authorized GitHub Apps = User access token.](../images/blog/single-github-app-for-multiple-accounts/github-app-tabs.gif)*Installed GitHub Apps = App access token. Authorized GitHub Apps = User access token.*

### User access token

The user access token allows us to perform actions on behalf of the user. We use this token for two operations:

* List the GitHub accounts that already installed our app, to which the authenticated user has access (check out the related [documentation](https://docs.github.com/en/rest/reference/apps#list-app-installations-accessible-to-the-user-access-token)).

* Create a repository on a personal GitHub account (check out the related [documentation](https://docs.github.com/en/rest/reference/repos#create-a-repository-for-the-authenticated-user)). GitHub doesn’t support creating repositories on personal accounts by using the App access token.

Before we can generate a user access token, the user must authorize our GitHub App. This can be done in two ways:

1. Explicitly ask the user to [authorize through GitHub](https://docs.github.com/en/developers/apps/authorizing-oauth-apps#1-request-a-users-github-identity) using the app’s client ID.

1. Request user authorization (OAuth) during app installation. To do so, manually enable this setting. It is located under the app “Identifying and authorizing users” section.

![App Settings / General / Identifying and authorizing users](../images/blog/single-github-app-for-multiple-accounts/request-authorization-github.png)*App Settings / General / Identifying and authorizing users*

In both ways, the user will be redirected to a URL you provided with a code as a query string parameter. Using the code, the app client-id and the appclient-secret we can generate the user access token.

Both the client-id and the client-secret located in the GitHub App page.

```ts
import axios from 'axios'

async function generateTokenFromCode(
 code: string,
 client_id: string,
 client_secret: string
) {
    const data = {
      client_id,
      client_secret,
      code
    }

    const res = await axios.post('https://github.com/login/oauth/access_token', data , {
      headers: {
        Accept: 'application/json'
      }
    })

    if (res?.data?.error === 'bad_verification_code') {
      throw new Error('Github server denied authorization code.')
    }

    return res
  }
```

You can read more about it in the [documentation](https://docs.github.com/en/developers/apps/identifying-and-authorizing-users-for-github-apps#2-users-are-redirected-back-to-your-site-by-github).

**Refresh Tokens**

[Refresh tokens](https://docs.github.com/en/developers/apps/refreshing-user-to-server-access-tokens#renewing-a-user-token-with-a-refresh-token) are supported as an optional feature. To enable it, go to the App settings / Optional features / User-to-server token expiration and click opt-in.

Be aware that each refresh token is for a single-use. Each time you use it, a new refresh token is generated and the previous one becomes invalid. It is crucial that you correctly store and manage the refresh token to avoid losing it, which would require you to request the authorization again.

### App access token

The App access token allows us to perform actions on behalf of the GitHub app. This access token is generated by providing the app-id, app-secret and the app installation-id for the GitHub account to which you request access.

```ts
import axios from 'axios'

async function generateAppAccessToken(appId: string, secret: string, installationId: string) {
   const now = Math.floor(Date.now() / 1000) - 30
    const expiration = now + 60 * 10 // JWT expiration time (10 minute maximum)

    const payload = {
      iat: now, // Issued at time
      exp: expiration,
      iss: appId
    }

    const jwt = jwt.sign(payload, secret, { algorithm: 'RS256' })

    const res = await axios.post(
        `https://api.github.com/app/installations/${installationId}/access_tokens`,
        {}, {
        headers: {
          Authorization: `Bearer ${jwt}`,
          Accept: 'application/vnd.github.machine-man-preview+json'
        }
      })

    return res
}
```

You can read more about creating app access tokens in the [documentation](https://docs.github.com/en/rest/reference/apps#create-an-installation-access-token-for-an-app), and also about an issue with expiration times, that we faced, [here](https://stackoverflow.com/questions/61770501/github-api-returns-401-while-trying-to-generate-access-token/61872948#61872948).

## Webhooks

Webhooks allow us to subscribe to GitHub App events. Using webhooks, we can react to events that happen to the GitHub App. For example, here are some of the webhooks that we use to manage the Altostra GitHub app:

1. App was installed — store the app installation information for future use.

1. App was removed — disconnect the accounts associated with the app and remove the installation information.

1. App authorization revoked — remove access token.

You can read more about webhooks [here](https://docs.github.com/en/developers/webhooks-and-events/webhook-events-and-payloads) and how to [secure](https://docs.github.com/en/developers/webhooks-and-events/securing-your-webhooks) them.

## Finally

Understanding the GitHub App workflows helped us provide our users with the best experience working with multiple teams and GitHub accounts.

Using the access tokens, webhooks, and APIs mentioned in this post, we created a robust solution to manage Altostra users’ GitHub accounts.

We intentionally avoided going into implementation details, as they are relevant only for us. We hope that the information above is enough to set you on the correct path to building your own solution.
