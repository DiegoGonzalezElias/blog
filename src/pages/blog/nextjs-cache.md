---
layout: "../../layouts/Post.astro"
title: Understanding Nextjs cache management
image: /images/posts/nextjs-cache/nextjs_cache
publishedAt: "2024-07-08"
category: "Nextjs"
---

## Introduction

Next.js is a widely used framework with a variety of utilities that enable more agile and efficient development. However, there is one topic that I find not very easy to understand, and that is how Next.js caches different things.

To better understand this, I should start by explaining that Next.js uses 4 different caches (Router cache, Full Route cache, Request Memoization cache, and Data cache). These are divided into 1 at the client level and 3 at the server level, as shown in the image.

![Imagen de ejemplo](/images/posts/nextjs-cache/nextjs_caches_chart.webp)

Throughout this post, we will discuss how each of these caches works, whether they are used by default in Next.js 14 and Next.js 15, how to interact with them, and how to bypass them.

### Request Memoization cache

This cache belongs to React, not to Next.js per se, but Next.js uses it to cache fetch requests. This means that if the same request is made on a page, it won’t be made twice (in case of two identical requests on the same page). Instead, the value of the first request is returned, having been cached.

At first glance, this is very logical because it prevents repeated requests and thus improves the efficiency of our application. However, there might be specific cases where we need to make the same request within a single page and not receive the cached value as the response for the second request.

If we want to avoid this behavior, we need to pass a "signal" object from "AbortController" in the request options.

```js
async function fetchCarById(id) {
  const { signal } = new AbortController();
  const res = await fetch(`https://myapi.com/cars/${id}`, {
    signal,
  });
  return res.json();
}
```

This cache is not used by default in Next.js 15, but it is in Next.js 14.

### Data cache

This cache is the last one that Next.js consults before fetching data from the API or database. It caches fetch requests so that when multiple users want to get the values of the same request, they are returned from the Data cache instead of making all the requests to the corresponding API/DB. This is especially useful for requests whose values change infrequently, thereby improving performance.

Let's return to the example of requesting car information by its ID.

```javascript
export default async function Page({ params }) {
  const id = params.id;
  const res = await fetch(`https://myapi.com/cars/${id}`);
  const car = await res.json();

  return (
    <div>
      <h1>{car.modelName}</h1>
      <p>{car.info}</p>
    </div>
  );
}
```

By using this cache, we can prevent each user wanting information about this vehicle from making a request to the API or database, as this information will not change much.

In Next.js 14, each fetch request in server components is stored in the Data cache by default. This does not occur in Next.js 15, as fetch requests are not cached by default in this version.

This cache is persistent even between deployments of the application. Therefore, it is necessary to know the two ways we can clear this cache. These are:

- **Time-based Revalidation**: This method specifies how often data in this cache is cleared. It can be done in two ways:

  1. The first option is by passing `next: { revalidate: 3600 }` in the fetch request options, indicating how long the response of this request will remain cached in the Data cache:

     ```javascript
     const res = fetch(`https://myapi.com/cars/${id}`, {
       next: { revalidate: 3600 },
     });
     ```

  2. The second option to set a revalidation time is to use the revalidate segment config option. This acts at the page level:

     ```javascript
     export const revalidate = 3600;

     export default async function Page({ params }) {
       const id = params.id;
       const res = await fetch(`https://myapi.com/cars/${id}`);
       const car = await res.json();

       return (
         <div>
           <h1>{car.modelName}</h1>
           <p>{car.info}</p>
         </div>
       );
     }
     ```

- **On-demand Revalidation**: If your data is not updated on a regular schedule, there's also the option to update the cached value when there's a change in this data. For example, in the case of a blog, it can use the cache unless there's a change and a new post is added.

  This can be done in two ways:

  1. Using `revalidatePath`, a function that takes the path from which you want to clear cached fetch request data:

     ```javascript
     import { revalidatePath } from "next/cache";

     export async function publishPost({ post }) {
       createPost(post);

       revalidatePath(`/posts/${post}`);
     }
     ```

  2. For more specific clearing, you can use the `revalidateTag` function.

     First, specify a tag in the fetch request options for the data you want to clear from the cache:

     ```javascript
     const res = fetch(`https://myapi.com/posts/${id}`, {
       next: { tags: ["post"] },
     });
     ```

     Then use the `revalidateTag` function with that tag to identify and clear that request's data from the cache:

     ```javascript
     import { revalidateTag } from "next/cache"

     export async function publishPost({ post }) {
       createPost(post)

       revalidateTag(‘post’)
     }
     ```

To bypass this cache, you can do so in two ways:

1. At the request level, by passing `'no-cache'` as the value to the `'cache'` property in the fetch request options:

   ```javascript
   const res = fetch(`https://myapi.com/blogs/${Id}`, {
     cache: "no-store",
   });
   ```

2.At the page level, either by converting the page to dynamic and adding the following line at the beginning of page.tsx:

```javascript
export const dynamic = "force-dynamic";
```

Or by also adding the revalidate segment config option at the beginning of the page, setting it equal to 0:

```javascript
export const revalidate = 0;
```

### Full Route cache

The main benefit of this cache is that it allows Next.js to cache static pages during compilation, avoiding the need to generate those static pages on every request. Specifically, it stores the HTML and RSCP (React Server Component Payload) files that make up the page.

Unlike the Data cache, this cache is cleared with each deployment.

You can avoid using this cache in two ways:

1. By avoiding the use of the Data cache. If Data cache is not used, the Full Route cache will also not be used.
2. The second way is by using dynamic data on your page.

### Router cache

This cache differs from the others we've seen in that it is stored on the client side rather than the server side. It caches the routes the user has visited to pull from this cached data (HTML and RSCP) instead of making a request to the server. By default in Next.js 14, if the route is static, it remains cached for 5 minutes, and if dynamic, it remains cached for 30 seconds. This cache is stored at the user's session level, so if the user refreshes the page or closes the tab, the cached data will be cleared.

We can revalidate the data in this cache using revalidatePath or revalidateTag just like we did with the Data cache. Therefore, if we clear the Data cache using these functions, we are also clearing the Router cache data.

## Example of behavior in Nextjs 14 and Nextjs 15

In this example, we are going to show the differences in behavior between versions 14 and 15 of Next.js using the same configuration in both projects. For this, we will need 3 projects.

1. One that sets up a very simple API with an endpoint that returns a random number.

![Imagen de ejemplo](/images/posts/nextjs-cache/nextjs-cache-1.webp)

2. A project in Next.js 14 and another in Next.js 15, each with a Home page and an About page.

![Imagen de ejemplo](/images/posts/nextjs-cache/nextjs-cache-2.webp)
![Imagen de ejemplo](/images/posts/nextjs-cache/nextjs-cache-3.webp)
![Imagen de ejemplo](/images/posts/nextjs-cache/nextjs-cache-4.webp)

As we can see, the difference between the Home and About pages is that the Home page retrieves the random number by calling the endpoint of the API we created, while on the About page, the random number is obtained through Math.random().

With these projects deployed, we will make a series of observations distinguishing between deployments in development and production mode.

![Imagen de ejemplo](/images/posts/nextjs-cache/nextjs-cache-5.webp)

### Development Mode

**Important**: To ensure these examples work, the Next.js `Link` component must be used for navigating between pages.

#### Version 14.2.4

In version 14.2.4 of Next.js, the first page that fetches data by calling an external API, regardless of attempts to navigate away and return to this page (home), or even refresh the page from the browser, will consistently return the same value. This indicates that the fetched value is being cached.

Conversely, on the About page, which displays a randomly generated number without making any external calls, refreshing or not will only yield a different/updated number when the page is refreshed from the browser. This behavior is due to Route Cache, which ensures that when navigating between pages and the maximum lifetime of the cached page has not expired, the cached page continues to be served.

#### Version 15.0-rc

In version 15.0.0-rc.0, the behavior becomes more intuitive, as it does not cache fetch requests by default nor uses Route Cache by default. Regardless of how one navigates through the different pages, accessing them will yield different values each time.

---

### Production Mode

#### Version 14.2.4

In production mode for version 14.2.4, no numbers change (neither on the home page nor on the about page). Only the about page number changes once when refreshed from the browser. This occurs because the page is cached in Route Cache.

#### Version 15.0-rc

The behavior in production mode for version 15.0-rc is identical to that of version 14.

---

## Solution

To address the issue of page caching in version 15, a directive can be applied at the top of each page to mark it as dynamic:

```javascript
export const dynamic = "force-dynamic";
```

This allows all these pages in version 15 to be built dynamically, preventing Route Cache application and functioning similarly to development mode.

However, in version 14, the only improvement achieved is that upon refreshing the About page from the browser, its value updates. Accessing it via the `Link` still returns the stored value in the Route Cache of the page. For the Home page, despite version 14 also caching fetch requests by default (unlike version 15), refreshing the page from the browser always yields the same value.
