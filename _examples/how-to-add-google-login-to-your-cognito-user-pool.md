---
layout: example
title: How to Add Google Login to Your Cognito User Pool
short_title: Google Auth
date: 2021-02-08 00:00:00
lang: en
index: 3
type: jwt-auth
description: In this example we will look at how to add Google Login to a Cognito User Pool using Serverless Stack (SST). We'll be using the sst.Api and sst.Auth to create an authenticated API.
short_desc: Authenticating a full-stack serverless app with Google.
repo: api-oauth-google
ref: how-to-add-google-login-to-your-cognito-user-pool
comments_id: how-to-add-google-login-to-your-cognito-user-pool/2643
---

In this example, we will look at how to add Google Login to Your Cognito User Pool using [Serverless Stack (SST)]({{ site.sst_github_repo }}).

## Requirements

- Node.js >= 10.15.1
- We'll be using Node.js (or ES) in this example but you can also use TypeScript
- An [AWS account]({% link _chapters/create-an-aws-account.md %}) with the [AWS CLI configured locally]({% link _chapters/configure-the-aws-cli.md %})
- A [Google API project](https://console.developers.google.com/apis)

## Create an SST app

{%change%} Let's start by creating an SST app.

```bash
$ npx create-serverless-stack@latest api-oauth-google
$ cd api-oauth-google
```

By default, our app will be deployed to an environment (or stage) called `dev` and the `us-east-1` AWS region. This can be changed in the `sst.json` in your project root.

```json
{
  "name": "api-oauth-google",
  "region": "us-east-1",
  "main": "stacks/index.js"
}
```

## Project layout

An SST app is made up of two parts.

1. `stacks/` — App Infrastructure

   The code that describes the infrastructure of your serverless app is placed in the `stacks/` directory of your project. SST uses [AWS CDK]({% link _chapters/what-is-aws-cdk.md %}), to create the infrastructure.

2. `src/` — App Code

   The code that's run when your API is invoked is placed in the `src/` directory of your project.

## Setting up the Auth

First, let's create a [Cognito User Pool](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-identity-pools.html) to store the user info using the [`Auth`](https://docs.serverless-stack.com/constructs/Auth) construct

{%change%} Add this code below the `super()` method in `stacks/MyStack.js`.

```js
// Create auth
const auth = new sst.Auth(this, "Auth", {
  cognito: {
    userPoolClient: {
      supportedIdentityProviders: [
        cognito.UserPoolClientIdentityProvider.GOOGLE,
      ],
      oAuth: {
        callbackUrls: ["http://localhost:3000"],
        logoutUrls: ["http://localhost:3000"],
      },
    },
  },
});
```

This creates a [Cognito User Pool](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-identity-pools.html); a user directory that manages users. We've configured the User Pool to allow users to login with their Google account and added the callback and logout URLs.

Note, we haven't yet set up Google OAuth with our user pool, we'll do it next.

## Setting up Google OAuth

Now let's add Google OAuth for our serverless app, to do so we need to create a [Google User Pool identity provider](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_cognito.UserPoolIdentityProviderGoogle.html) and link it with the user pool we created above.

{%change%} Create a `.env` file in the root and add your google `clientId` and `clientSecret` from your [Google API project](https://console.developers.google.com/apis).

```js
GOOGLE_CLIENT_ID=<YOUR_GOOGLE_CLIENT_ID>
GOOGLE_CLIENT_SECRET=<YOUR_GOOGLE_CLIENT_SECRET>
```

{%change%} Add this below the `sst.Auth` definition in `stacks/MyStack.js`.

```js
// Create a Google OAuth provider
const provider = new cognito.UserPoolIdentityProviderGoogle(this, "Google", {
  clientId: process.env.GOOGLE_CLIENT_ID,
  clientSecret: process.env.GOOGLE_CLIENT_SECRET,
  userPool: auth.cognitoUserPool,
  scopes: ["profile", "email", "openid"],
  attributeMapping: {
    email: cognito.ProviderAttribute.GOOGLE_EMAIL,
    givenName: cognito.ProviderAttribute.GOOGLE_GIVEN_NAME,
    familyName: cognito.ProviderAttribute.GOOGLE_FAMILY_NAME,
    profilePicture: cognito.ProviderAttribute.GOOGLE_PICTURE,
  },
});

// attach the created provider to our userpool
auth.cognitoUserPoolClient.node.addDependency(provider);
```

This creates a Google identity provider with the given scopes and links the created provider to our user pool.

Make sure to import the `cognito` package.

```js
import * as cognito from "aws-cdk-lib/aws-cognito";
```

Now let's associate a Cognito domain to the user pool, which we can be used for sign-up and sign-in webpages.

{%change%} Add this below code in `stacks/MyStack.js`.

```js
// Create a cognito userpool domain
const domain = auth.cognitoUserPool.addDomain("AuthDomain", {
  cognitoDomain: {
    domainPrefix: `${scope.stage}-demo-auth-domain`,
  },
});
```

## Setting up the API

{%change%} Replace the `sst.Api` definition with the following in `stacks/MyStacks.js`.

```js
// Create a HTTP API
const api = new sst.Api(this, "Api", {
  defaultAuthorizer: new apigAuthorizers.HttpUserPoolAuthorizer(
    "Authorizer",
    auth.cognitoUserPool,
    {
      userPoolClients: [auth.cognitoUserPoolClient],
    }
  ),
  defaultAuthorizationType: sst.ApiAuthorizationType.JWT,
  routes: {
    "GET /private": "src/private.handler",
    "GET /public": {
      function: "src/public.handler",
      authorizationType: sst.ApiAuthorizationType.NONE,
    },
  },
});

// Allow authenticated users invoke API
auth.attachPermissionsForAuthUsers([api]);
```

We are creating an API here using the [`sst.Api`](https://docs.serverless-stack.com/constructs/api) construct. And we are adding two routes to it.

```
GET /private
GET /public
```

By default, all routes have the authorization type `JWT`. This means the caller of the API needs to pass in a valid JWT token. The first is a private endpoint. The second is a public endpoint and its authorization type is overridden to `NONE`.

Let's install the npm packages we are using here.

{%change%} From the project root run the following.

```bash
$ npx sst add-cdk @aws-cdk/aws-apigatewayv2-authorizers-alpha
```

The reason we are using the [**add-cdk**](https://docs.serverless-stack.com/packages/cli#add-cdk-packages) command instead of using an `npm install`, is because of [a known issue with AWS CDK](https://docs.serverless-stack.com/known-issues). Using mismatched versions of CDK packages can cause some unexpected problems down the road. The `sst add-cdk` command ensures that we install the right version of the package.

## Adding function code

Let's create two functions, one handling the public route, and the other for the private route.

{%change%} Add a `src/public.js`.

```js
export async function handler() {
  return {
    statusCode: 200,
    body: "Hello, stranger!",
  };
}
```

{%change%} Add a `src/private.js`.

```js
export async function handler() {
  return {
    statusCode: 200,
    body: "Hello, user!",
  };
}
```

## Setting up our React app

To deploy a React app to AWS, we'll be using the SST [`ViteStaticSite`](https://docs.serverless-stack.com/constructs/ViteStaticSite) construct.

{%change%} Replace the `this.addOutputs` call with the following.

```js
// Create a React Static Site
const site = new sst.ViteStaticSite(this, "Site", {
  path: "frontend",
  environment: {
    VITE_APP_COGNITO_DOMAIN: domain.domainName,
    VITE_APP_API_URL: api.url,
    VITE_APP_REGION: scope.region,
    VITE_APP_USER_POOL_ID: auth.cognitoUserPool.userPoolId,
    VITE_APP_IDENTITY_POOL_ID: auth.cognitoCfnIdentityPool.ref,
    VITE_APP_USER_POOL_CLIENT_ID: auth.cognitoUserPoolClient.userPoolClientId,
  },
});

// Show the endpoint in the output
this.addOutputs({
  ApiEndpoint: api.url,
  authClientId: auth.cognitoUserPoolClient.userPoolClientId,
  domain: domain.domainName,
  site_url: site.url,
});
```

The construct is pointing to where our React.js app is located. We haven't created our app yet but for now, we'll point to the `frontend` directory.

We are also setting up [build time React environment variables](https://vitejs.dev/guide/env-and-mode.html) with the endpoint of our API. The [`ViteStaticSite`](https://docs.serverless-stack.com/constructs/ViteStaticSite) allows us to set environment variables automatically from our backend, without having to hard code them in our frontend.

We are going to print out the resources that we created for reference.

## Starting your dev environment

{%change%} SST features a [Live Lambda Development](https://docs.serverless-stack.com/live-lambda-development) environment that allows you to work on your serverless apps live.

```bash
$ npx sst start
```

The first time you run this command it'll take a couple of minutes to deploy your app and a debug stack to power the Live Lambda Development environment.

```
===============
 Deploying app
===============

Preparing your SST app
Transpiling source
Linting source
Deploying stacks
manitej-api-oauth-google-my-stack: deploying...

 ✅  manitej-api-oauth-google-my-stack


Stack manitej-api-oauth-google-my-stack
  Status: deployed
  Outputs:
    ApiEndpoint: https://sez1p3dsia.execute-api.ap-south-1.amazonaws.com
    SiteUrl: https://d2uyljrh4twuwq.cloudfront.net
```

The `ApiEndpoint` is the API we just created. While the `SiteUrl` is where our React app will be hosted. For now, it's just a placeholder website.

Let's test our endpoint with the [SST Console](https://console.serverless-stack.com). The SST Console is a web based dashboard to manage your SST apps. [Learn more about it in our docs]({{ site.docs_url }}/console).

Go to the **API** tab and click **Send** button of the `GET /public` to send a `GET` request.

Note, The [API explorer]({{ site.docs_url }}/console#api) lets you make HTTP requests to any of the routes in your `Api` construct. Set the headers, query params, request body, and view the function logs with the response.

![API explorer invocation response](/assets/examples/api-oauth-google/api-explorer-invocation-response.png)

You should see a `Hello, stranger!` in the response body.

## Creating the frontend

Run the below commands in the root to create a basic react project.

```bash
# npm 7+, extra double-dash is needed:
$ npm init vite@latest frontend -- --template react

$ cd frontend

$ npm install
```

This sets up our React app in the `frontend/` directory. Recall that, earlier in the guide we were pointing the `ViteStaticSite` construct to this path.

We also need to load the environment variables from our SST app. To do this, we'll be using the [`@serverless-stack/static-site-env`](https://www.npmjs.com/package/@serverless-stack/static-site-env) package.

{%change%} Install the `static-site-env` package by running the following in the `frontend/` directory.

```bash
$ npm install @serverless-stack/static-site-env --save-dev
```

We need to update our start script to use this package.

{%change%} Replace the `dev` script in your `frontend/package.json`.

```bash
"dev": "vite"
```

{%change%} With the following:

```bash
"dev": "sst-env -- vite"
```

## Adding AWS Amplify

To use our AWS resources on the frontend we are going to use [AWS Amplify](https://aws.amazon.com/amplify/).

Note, to know more about configuring Amplify with SST check [this chapter]({% link _chapters/configure-aws-amplify.md %})

Run the below command to install AWS Amplify in the `frontend/` directory.

```bash
npm install aws-amplify
```

{%change%} Replace `src/main.jsx` with below code.

```js
/* eslint-disable no-undef */
import React from "react";
import ReactDOM from "react-dom";
import "./index.css";
import App from "./App";
import Amplify from "aws-amplify";

// Configure AWS Amplify with credentials from backend
Amplify.configure({
  Auth: {
    region: import.meta.env.VITE_APP_REGION,
    userPoolId: import.meta.env.VITE_APP_USER_POOL_ID,
    userPoolWebClientId: import.meta.env.VITE_APP_USER_POOL_CLIENT_ID,
    mandatorySignIn: false,
    oauth: {
      domain: `${
        import.meta.env.VITE_APP_COGNITO_DOMAIN +
        ".auth." +
        import.meta.env.VITE_APP_REGION +
        ".amazoncognito.com"
      }`,
      scope: ["email", "profile", "openid", "aws.cognito.signin.user.admin"],
      redirectSignIn: "http://localhost:3000", // Make sure to use the exact URL
      redirectSignOut: "http://localhost:3000", // Make sure to use the exact URL
      responseType: "token", // or 'token', note that REFRESH token will only be generated when the responseType is code
    },
  },
  API: {
    endpoints: [
      {
        name: "api",
        endpoint: import.meta.env.VITE_APP_API_URL,
        region: import.meta.env.VITE_APP_REGION,
      },
    ],
  },
});

ReactDOM.render(
  <React.StrictMode>
    <App />
  </React.StrictMode>,
  document.getElementById("root")
);
```

## Adding login UI

{%change%} Replace `src/App.jsx` with below code.

{% raw %}
```jsx
import { Auth, API } from "aws-amplify";
import React, { useState, useEffect } from "react";

const App = () => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  // Get the current logged in user info
  const getUser = async () => {
    const user = await Auth.currentUserInfo();
    if (user) setUser(user);
    setLoading(false);
  };

  // Trigger Google login
  const signIn = async () =>
    await Auth.federatedSignIn({
      provider: "Google",
    });

  // Logout the authenticated user
  const signOut = async () => await Auth.signOut();

  // Send an API call to the /public endpoint
  const publicRequest = async () => {
    const response = await API.get("api", "/public");
    alert(JSON.stringify(response));
  };

  // Send an API call to the /private endpoint with authentication details.
  const privateRequest = async () => {
    try {
      const response = await API.get("api", "/private", {
        headers: {
          Authorization: `Bearer ${(await Auth.currentSession())
            .getAccessToken()
            .getJwtToken()}`,
        },
      });
      alert(JSON.stringify(response));
    } catch (error) {
      alert(error);
    }
  };

  // Check if there's any user on mount
  useEffect(() => {
    getUser();
  }, []);

  if (loading) return <div className="container">Loading...</div>;

  return (
    <div className="container">
      <h2>SST + Cognito + Google OAuth + React</h2>
      {user ? (
        <div className="profile">
          <p>Welcome {user.attributes.given_name}!</p>
          <img
            src={user.attributes.picture}
            style={{ borderRadius: "50%" }}
            width={100}
            height={100}
            alt=""
          />
          <p>{user.attributes.email}</p>
          <button onClick={signOut}>logout</button>
        </div>
      ) : (
        <div>
          <p>Not signed in</p>
          <button onClick={signIn}>login</button>
        </div>
      )}
      <div className="api-section">
        <button onClick={publicRequest}>call /public</button>
        <button onClick={privateRequest}>call /private</button>
      </div>
    </div>
  );
};

export default App;
```
{% endraw %}

{%change%} Replace `src/index.css` with the below styles.

```css
body {
  margin: 0;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", "Roboto",
    "Oxygen", "Ubuntu", "Cantarell", "Fira Sans", "Droid Sans",
    "Helvetica Neue", sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

code {
  font-family: source-code-pro, Menlo, Monaco, Consolas, "Courier New",
    monospace;
}

.container {
  width: 100%;
  height: 100vh;
  display: flex;
  flex-direction: column;
  align-items: center;
  text-align: center;
}

button {
  width: 120px;
  padding: 10px;
  border: none;
  border-radius: 4px;
  background-color: #000;
  color: #fff;
  font-size: 16px;
  cursor: pointer;
}

.profile {
  border: 1px solid #ccc;
  padding: 20px;
  border-radius: 4px;
}
.api-section {
  width: 100%;
  margin-top: 20px;
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 10px;
}

.api-section > button {
  background-color: darkorange;
}
```

Let's start our frontend in development environment.

{%change%} In the `frontend/` directory run.

```bash
npm run dev
```

Open up your browser and go to `http://localhost:3000`.

![Browser view of localhost](/assets/examples/api-oauth-google/browser-view-of-localhost.png)

There are 2 buttons that invokes the endpoints we created above.

The **call /public** button invokes **GET /public** route using the `publicRequest` method we created in our frontend.

Similarly, the **call /private** button invokes **GET /private** route using the `privateRequest` method.

When you're not logged in and try to click the buttons, you'll see responses like below.

![public button click without login](/assets/examples/api-oauth-google/public-button-click-without-login.png)

![private button click without login](/assets/examples/api-oauth-google/private-button-click-without-login.png)

Once you click on login, you're asked to login through your Google account. Once it's done you can check your info.

![current logged in user info](/assets/examples/api-oauth-google/current-logged-in-user-info.png)

Now that you've authenticated repeat the same steps as you did before, you'll see responses like below.

![public button click with login](/assets/examples/api-oauth-google/public-button-click-with-login.png)

![private button click with login](/assets/examples/api-oauth-google/private-button-click-with-login.png)

As you can see the private route is only working while we are logged in.

## Deploying your API

{%change%} To wrap things up we'll deploy our app to prod.

```bash
$ npx sst deploy --stage prod
```

This allows us to separate our environments, so when we are working in `dev`, it doesn't break the app for our users.

Once deployed, you should see something like this.

```bash
 ✅  prod-api-oauth-google-my-stack


Stack prod-api-oauth-google-my-stack
  Status: deployed
  Outputs:
    ApiEndpoint: https://ck198mfop1.execute-api.us-east-1.amazonaws.com
```

## Cleaning up

Finally, you can remove the resources created in this example using the following command.

```bash
$ npx sst remove
```

And to remove the prod environment.

```bash
$ npx sst remove --stage prod
```

## Conclusion

And that's it! You've got a brand new serverless API authenticated with Google. A local development environment, to test. And it's deployed to production as well, so you can share it with your users. Check out the repo below for the code we used in this example. And leave a comment if you have any questions!