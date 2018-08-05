---
title: "Configuring JavaScript Web Applications"
date: 2018-08-06T08:29:13+10:00
slug: "web-app-config"
description: ""
tags: ["web", "configuration", "react"]
type: "post"
---

_I wrote the post below almost a year ago in October 2017, but it's still the approach that I tend to use so I think it's worth posting here. I've made some minor edits for clarity._

---

When creating a web site with meaningful functionality (not just static pages) it's currently very popular to use a [Single Page Application (SPA)](https://en.wikipedia.org/wiki/Single-page_application) approach. These SPAs are mostly written in Javascript (or something that compiles to it). To handle the complexity that's involved (bundling, minifying, etc.) usually something like [WebPack](https://webpack.github.io/) is also used to help with development.

My current preferred approach is to use [React](https://facebook.github.io/react/), usually via [`create-react-app`](https://github.com/facebookincubator/create-react-app) (or more typically, [its TypeScript equivalent](https://github.com/Microsoft/TypeScript-React-Starter)) in order to make setting up the development pipeline as simple as possible.

But as with software development in general, these web applications inevitably require some degree of configuration to get them running correctly.

## What I mean by "configuration" ...

When a web application is built, usually there are at least a few "values" that are needed by the application to function, and generally they do not change during a usage session. These may be things like:

- A key to be used for an Analytics service
- The base url for an API server
- The logging level (debug, info, warning, etc) to use

However, these values should not necessarily be all treated the same way:

- Some may be safely public, others should be (more) secret
- Some will rarely or never change between sessions, others may change more frequently
- Some should be the same between environments (development, staging, production), others will need to change

## Right, so what are some options?

There are a number of ways that such values can be provided to the application, each with pros and cons.

### Values fetched from the server

This approach is that when the web application is loaded by the browser, a call is made to a server API to retrieve the configuration values. This effectively delegates the responsibility for config values to the server.

The pros of this approach are:

- It generally makes configuration very simple from a front-end point of view - it just needs to make an API call and then hold the results somewhere appropriate

The cons of this approach are:

- It requires a reachable API server (no good for offline unless the values can be cached)
- If the API is hosted at a different domain to the web app (ie. cross origin), then the URL of the API needs to be known in order to make the call - so we still have a config situation to solve!
- Getting the configuration becomes an async action (this may or may not be a problem)

I generally find that this is a good approach if the API is (and always will be!) hosted on the same domain as the web app, as servers generally have well-estabished ways of handling configuration. But if this is not the case, read on!

### Hardcoded values

Why not just put the config values straight into the code of the web app? This is a fine approach, if your situation can handle the following caveats:

- _Values are not secret_ - committing any secret values (such as keys) to source control is a bad idea
- _Values will rarely or never change_ - and if they do change, you're happy to make a code change and run through your entire build and release process
- _Values do not need to change per environment_ - the value is the same during development as it is in production

Unfortunately I rarely (if ever) have configuration values that can fulfil these requirements, so what else is there?

### Values swapped in at build time

With this approach, our configuration values are set as part of the build of the web app, often via [Environment Variables](https://en.wikipedia.org/wiki/Environment_variable). This allows having certain values when developing locally, and different values when we build for releasing.

With `create-react-app` I can create a file such as `appSettings.js` containing something like:

```js
export const NODE_ENV = process.env.NODE_ENV;
export const API_BASE_URL = (process.env.REACT_APP_API_BASE_URL || "") + "/";
```

and the `process.env.XXXX` parts will be automatically replaced by WebPack. Other build tools should have something similar available.

Using this approach solves some (but not all) of the limitations that hardcoded values have:

- _We can now have "secret" values_ - because the built artifacts are not checked into source control, and the config values can be securely provided by a build server. But keep in mind that they'll still be sent to the client at run time (embedded in the js) so they won't be entirely secret.
- _We can now change a value without a code change_ - but we will need a rebuild
- _Local development-time values can be provided_ - `create-react-app` uses [`.env` file(s)](https://github.com/facebookincubator/create-react-app/blob/master/packages/react-scripts/template/README.md#adding-custom-environment-variables) to set custom variables but they can also be easily set at the command line for a single session if needed, which can be added to a local run script.
- _We still **can't** change our values per environment_ - we are still unable to deploy a build to a test environment and then to a production environment without having to rebuild, as the build-time values are captured directly into the build artifacts

This approach can handle many situations, especially if the release pipeline is relatively simple (such as only having a single environment). It is also quite simple to manage.

But if you do have environment-specific values...

### Values swapped in at deployment time

The final approach is mostly to allow having different values for different environments. For example: in the Test environment, the test API server (at `https://test.example.com`) should be called, but in the Production environment the prod API server (at `https://example.com`) should be called instead.

To achieve this, I've started using the following approach:

1.  When building the web app, set the environment variable to something that can be swapped out using a find-and-replace approach during deployment.

    - I've been using [Visual Studio Team Services (VSTS)](https://www.visualstudio.com/team-services/) lately, and it has a build/release task called [Replace Tokens](https://marketplace.visualstudio.com/items?itemName=qetza.replacetokens). So, instead of setting the value to something like "`https://example.com`", set it to something like "`#{REACT_APP_API_BASE_URL}#`" (the "#{ }#" parts are what Replace Tokens looks for).

2.  During deploy, run a find-and-replace over the relevant js file, and replace it with the value that you need for the particular environment.

    - So for VSTS, it's a matter of adding a Replace Tokens deploy task, and setting variables (such as "REACT_APP_API_BASE_URL") with appropriate values and scoping them to the relevant environment. For `create-react-app`, my `appSettings.js` gets bundled into the "main" package and has a cache-busting hash suffixed. So the replace task targets the `main.*.js` file in the js folder.

This approach does mean scanning though a potentially large js file looking for substitutions, but the special characters used to flag the replacement values should be unique enough so as to not risk replacing something unintended.

## Wrap up

When retrieving configuration values from an API at runtime is not an option, and neither is hardcoding values, environment variables are a flexible way of providing development-time, build-time and deployment-time configurations.

- They're simple to understand
- They're generally simple to use due to the standard support for using environment variables in build tools such as WebPack and VSTS.
- The developer can use development values, and then choose whether to swap out configuration values at build time _or_ deployment time, without having to change any code - it's all configured as part of the build and deployment steps.

May your configurations be as simple as possible!
