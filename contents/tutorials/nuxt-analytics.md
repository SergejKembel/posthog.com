---
title: How to set up analytics in Nuxt
date: 2024-01-19
featuredVideo: https://www.youtube-nocookie.com/embed/7-GS9srUsqs
author:
  - lior-neu-ner
tags:
  - product analytics
---

[Product analytics](/product-analytics) enable you to gather and analyze data about how users interact with your Nuxt.js app. To show you how to set up analytics, in this tutorial we create a basic Nuxt app, add PostHog on both the client and server, and use it to capture pageviews and custom events.

## Creating a Nuxt app

To demonstrate the basics of PostHog analytics, we create a simple Nuxt 3 app with two pages and a link to navigate between them.

First, ensure [Node.js is installed](https://nodejs.dev/en/learn/how-to-install-nodejs/) (version 18.0.0 or newer). Then run the following command:

```bash
npx nuxi@latest init <project-name>
```

Name it whatever you like (we call ours `nuxt-analytics`), select `npm` as the package manager, and use the defaults for the remaining options.

### Adding pages

Next, create a new directory `pages` and two new files `index.vue` and `about.vue` in it:

```bash
cd ./nuxt-analytics # replace "nuxt-analytics" with your project name
mkdir pages
cd ./pages
touch index.vue
touch about.vue
```

In `index.vue`, add the following the code:

```vue file=pages/index.vue
<template>
  <div>
    <h1>Home</h1>
    <NuxtLink to="/about">Go to About</NuxtLink>
  </div>
</template>
```

In `about.vue`, add the following the code:

```vue file=pages/about.vue
<template>
  <div>
    <h1>About</h1>
    <NuxtLink to="/">Back Home</NuxtLink>
  </div>
</template>
```

Lastly, replace the code in `app.vue` with:

```vue file=app.vue
<template>
  <div>
    <NuxtLayout>
      <NuxtPage/>
    </NuxtLayout>
  </div>
</template>
```

The basic setup is now complete. Run `npm run dev` to see your app.

![Basic Nuxt app](https://res.cloudinary.com/dmukukwp6/image/upload/v1710055416/posthog.com/contents/images/tutorials/nuxt-analytics/basic-app.png)

## Adding PostHog on the client side

> This tutorial shows how to integrate PostHog with `Nuxt 3`. If you're using `Nuxt 2`, see [our Nuxt docs](/docs/libraries/nuxt-js) for how to integrate.

With our app set up, it’s time to install and set up PostHog. If you don't have a PostHog instance, you can [sign up for free](https://us.posthog.com/signup). 

First install `posthog-js`:

```bash
npm install posthog-js
```

Then, add your PostHog API key and host to your `nuxt.config.ts` file. You can find your project API key in your [PostHog project settings](https://app.posthog.com/settings/project).

```ts file=nuxt.config.ts
export default defineNuxtConfig({
 devtools: { enabled: true },
  runtimeConfig: {
    public: {
      posthogPublicKey: '<ph_project_api_key>',
      posthogHost: '<ph_client_api_host>',
      posthogDefaults: '<ph_posthog_js_defaults>',
    }
  }
})
```

Create a new [plugin](https://nuxt.com/docs/guide/directory-structure/plugins) by creating a new folder called `plugins` in your base directory and then a new file `posthog.client.js`:

```bash
mkdir plugins
cd plugins 
touch posthog.client.js
```

Add the code initialize PostHog to your `posthog.client.js` file.

```js file=plugins/posthog.client.js
import { defineNuxtPlugin } from '#app'
import posthog from 'posthog-js'

export default defineNuxtPlugin(nuxtApp => {
  const runtimeConfig = useRuntimeConfig();
  const posthogClient = posthog.init(runtimeConfig.public.posthogPublicKey, {
    api_host: runtimeConfig.public.posthogHost,
    defaults: runtimeConfig.public.posthogDefaults,
  })
  
  return {
    provide: {
      posthog: () => posthogClient
    }
  }
})
```

Once you’ve done this, reload your app. You should begin seeing events in the [PostHog events explorer](https://us.posthog.com/events).

<ProductScreenshot
  imageLight="https://res.cloudinary.com/dmukukwp6/image/upload/Clean_Shot_2025_05_22_at_14_55_19_2x_483d9192c7.png" 
  imageDark="https://res.cloudinary.com/dmukukwp6/image/upload/Clean_Shot_2025_05_22_at_14_55_38_2x_b61b132048.png" 
  alt="Events in PostHog" 
  classes="rounded"
/>

### Capturing custom events

Beyond pageviews, there might be more events you want to capture. You can do this by capturing custom events with PostHog. 

To showcase this, we add a button to the home page and capture a custom event whenever it is clicked. Update the code in `index.vue`:

```vue file=index.vue
<template>
  <div>
    <h1>Home</h1>
    <NuxtLink to="/about">Go to About</NuxtLink>
    <button @click="captureCustomEvent">Click Me</button>
  </div>
</template>

<script setup>
const { $posthog } = useNuxtApp()

const captureCustomEvent = () => {
  $posthog().capture('home_button_clicked', {
    'user_name': 'Max the Hedgehog' 
  });
};
</script>
```

When you click the button now, PostHog captures a custom `home_button_clicked` event. Notice that we also added a property `user_name` to the event. This is helpful for filtering events in the PostHog dashboard.

## Adding PostHog on the server side

Nuxt is a full stack framework, parts of it run on both the client and server side. So far, we’ve only used PostHog on the client side and the [`posthog-js` library](/docs/libraries/js) we installed won’t work on the server side. Instead, we must use the [`posthog-node` SDK](/docs/libraries/node). We can start by installing it:

```bash
npm install posthog-node
```

Next, initialize `posthog-node` wherever you'd like to capture events server-side, such as in your [server routes](https://nuxt.com/docs/guide/directory-structure/server) or in the [middleware](https://nuxt.com/docs/guide/directory-structure/server#server-middleware). For this tutorial, we'll show you how to capture an event in the middleware:

Create a folder `middleware` in `server` directory and then a new file `analytics.js` in it:

```bash
cd ./server
mkdir middleware
cd ./middleware
touch analytics.js
```

Inside `analytics.js`, create a middleware function. Initialize PostHog in it and capture an event:

```js file=server/middleware/analytics.js
import { PostHog } from 'posthog-node';

export default defineEventHandler(async (event) => {
  const runtimeConfig = useRuntimeConfig();
  const posthog = new PostHog(
    runtimeConfig.public.posthogPublicKey,
    { host: runtimeConfig.public.posthogHost }
  );

  posthog.capture({
    distinctId: 'placeholder_distinct_id_of_the_user', 
    event: 'in_the_middleware',
  });
  await posthog.shutdown()  
});
```

> **Note**: Make sure to _always_ call `await posthog.shutdown()` after capturing events from the server-side.
> PostHog queues events into larger batches, and this call forces all batched events to be flushed immediately.

If you run your app again, you should begin to see `in_the_middleware` events in PostHog.

### Handling `distinctId` on the server

You may have noticed that we set `distinctId: placeholder_distinct_id_of_the_user` in our event above. To attribute events to the correct user, `distinctId` should be set to your user's unique ID. For logged-in users, we typically use their email as their `distinctId`. However, for logged-out users, you can use the `distinct_id` property from their PostHog cookie:

```js file=server/middleware/analytics.js
import { PostHog } from 'posthog-node';

export default defineEventHandler(async (event) => {
  const runtimeConfig = useRuntimeConfig();
  const cookieString =  event.node.req.headers.cookie || '';  
  const cookieName =`ph_${runtimeConfig.public.posthogPublicKey}_posthog`;
  const cookieMatch = cookieString.match(new RegExp(cookieName + '=([^;]+)'));
  
  let distinctId;
  if (cookieMatch) {
    const parsedValue = JSON.parse(decodeURIComponent(cookieMatch[1]));
    if (parsedValue && parsedValue.distinct_id) {
      distinctId = parsedValue.distinct_id;
      const posthog = new PostHog(
        runtimeConfig.public.posthogPublicKey,
        { host: runtimeConfig.public.posthogHost }
      );
      posthog.capture({
        distinctId: distinctId, 
        event: 'in_the_middleware',
      });
      await posthog.shutdown()
    } 
  }
});
```

## Further reading

- [How to set up feature flags in Nuxt](/tutorials/nuxt-feature-flags)
- [How to set up A/B tests in Nuxt](/tutorials/nuxtjs-ab-tests)
- [How to set up surveys in Nuxt](/tutorials/nuxt-surveys)

<NewsletterForm />