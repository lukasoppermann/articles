---
title: Making your website fast & offline available with a service worker
author: Lukas Oppermann
series: Building a Progressive web app;3
category: code
description: Iteratively turning a PHP base website into a progressive web app
preview: Learn how to start transforming your website into a progressive web app. In part three we use gulp to build your service worker.
---

> Let's see how a service worker will let us control the network, a super power with which we can make our page offline available and control caching for super fast loads.

In the <a href="170117-building-a-pwa-from-your-existing-website" class="o-link o-link--decorated">previous parts of this series</a>
we started converting our php based website into a progressive web app (PWA) by moving to easily cacheable & fast static files created with gulp and mustache.

Now we move one step further and cache all those files on the first visit which speeds up any further visit quite a lot.

## Understanding service workers

A service worker is like a very small middleware that sits between the client and the server. Any request from your website goes through the service worker. The service worker can than decide how to respond to the request. It could for example send  something back from the cache or forwarding the request to the server. The service worker can also try to get an answer from the server and only send back a cached response if the server fails, like in the image below.

<img class="o-article__image" src="/blog/media/pwa-part-3-service-worker.svg" alt="service worker explanation" />

**Note:** Because a service worker could potentially be used for a man-in-the-middle attack, you need to be on a secure ssl connection to use service workers.

### Service worker life cycle

The service worker has a life cycle separated from the web pages. This means it loads in the background and does **NOT** slow down your initial load or delay rendering (if you load it in a non render blocking manner).

#### Installation
Once the service worker is registered via your page's javascript, the install event is fired. You may choose to prefetch & cache files during installation or ignore the event all together. Typically you will want to cache some static assets, like the main css and js files. Be careful with the files you request though, because if any of the files are not available the installation will fail and the service worker won't activate.

#### Activation
If the installation was successful the service worker will activate. During the activation step you normally deal with invalidating old caches.

#### Idle
Once the activation is done, the service worker sits there idle, waiting for a request to happen. Using the `fetch` event we can deal with requests in a manner we choose, like returning a cached response.

If you wish to learn more check out this article about the <a href="https://developers.google.com/web/fundamentals/getting-started/primers/service-workers" class="o-link o-link--decorated">theory behind service workers</a>.

## Setting up your service worker

While it is entirely possible, to write our service worker from scratch, we will use a package by our friends at google, called [`sw-precache`](https://github.com/GoogleChrome/sw-precache) to do all the heavy lifting.

```bash
# adding the package to our dev dependencies via yarn
$ yarn add -D sw-precache
```

A service worker needs two files, to work. The actual `service-worker.js` file and a file to register the service worker. The latter is a lot of boiler plate code, like checking if service workers are supported. Luckily there is a pre-made file from the `sw-precache` folks, that we can just drop in: <a href="https://github.com/GoogleChrome/sw-precache/blob/master/demo/app/js/service-worker-registration.js" target="_blank" class="o-link o-link--decorated">service-worker-registration.js</a>

### The registration script

In this script, we check if the browser supports service workers `if ('serviceWorker' in navigator)` (<a href="http://caniuse.com/#search=serviceworker" class="o-link o-link--decorated" target="_blank">currently Firefox, Chrome & Opera</a>). If it does, once the `load` event fires the service worker is registered by the script `navigator.serviceWorker.register('service-worker.js')` and message is logged: either that the content has been updated or an error occurred, quite handy for debugging.

Because service workers are made to enhance the normal online experience, browsers which do not support service workers will just get the normal website straight from your server.

Make sure to <a href="170327-building-a-pwa-from-your-existing-website-part-2">revision this file</a> as well, so that it will be cache-busted if you change something.

Now you only need to load this script on your website and it will register the `service-worker.js` file for you.

### Creating the service worker

One important thing to know about the `service–worker.js` script is, that it can only control request in its root directory and below. This means you will want to put it into your websites `root` directory, so that the service worker can control all of your pages traffic.

We want `gulp` to build our service worker so we need to create a new `service-worker` task. Of course we need to require the `sw-precache` package. Additionally we define the `rootDir` and the urls we want to prefetch. Be careful with prefetching, even though it does not block downloading, it will still cost your users bandwidth, so don't just download your entire website.

```js
/* ------------------------------
 *
 * service-worker
 *
 */
gulp.task('service-worker', function (done) {
  const swPrecache = require('sw-precache')
  const rootDir = 'public'
  // urls to prefetch
  let urlsToPrefetch = [
    'media/veare-icons@2x.png',
    'css/app.css'
  ]
  /* more to come here */
})
```

You might remember, that we are using revisioned files, meaning the `app.css` is actually something like `app-123ldsajflkdsaj13.css`. Obviously you do not want to update this reference by hand, every time you change your assets. To work around this, we need to check which of the files have a revisioned version in our `rev-manifest.json` and update the array accordingly. For this we add the following below the `urlsToPrefetch` array.

```js
// get revisioned file version if exists in manifest
const fileHashes = JSON.parse(fs.readFileSync(`${rootDir}/rev-manifest.json`, 'utf8'))
// replace url with revisioned url in urlsToPrefetch
urlsToPrefetch = urlsToPrefetch.map(function (item) {
  // replace item with revisioned item
  if (typeof fileHashes[item] !== 'undefined') {
    return `${rootDir}/${fileHashes[item]}`
  }
  // return item if no revision is available
  return `${rootDir}/${item}`
})
```

Okay, what are we doing? First we load the `rev-manifest.json` and parse it, so we get an `object`. Afterwards we iterate through the key-value pairs and check if one matches our hash-suffixed filename. If it does, we replace the value in our object with the revisioned url, otherwise we return the non-revisioned one. Additionally we need to add the `rootDir` to the value, as it is not specified in our `rev-manifest`.

### SW-Precache
Now we can get into `sw-precache`, add the following below the `urlsToPrefetch.map` call:

```js
swPrecache.write(`${rootDir}/service-worker.js`, {
  staticFileGlobs: urlsToPrefetch,
  stripPrefix: rootDir,
  runtimeCaching: [{
    urlPattern: '/(.*)',
    handler: 'cacheFirst'
  },{
    urlPattern: /\.googleapis\.com\//,
    handler: 'cacheFirst'
  }]
}, done)
```

The `sw-precache.write` method has the signature `write(filePath, options, callback)`. It generates a service worker and writes it the the `filePath` taking into consideration the supplied `options`. Once it is done, the `callback` is invoked, either with an `error` parameter or `null`, if no error occurred.

#### filePath
The `filePath` is simply `service-worker.js` in the root directory: `${rootDir}/service-worker.js`.

#### options
For the options object, we supply our `urlsToPrefetch` array to `staticFileGlobs`. This will prefetch those files and make them available in the cache.

`stripPrefix: rootDir` removes the root directory (whatever you stored in the `rootDir` constant) from our urls when caching at runtime, which may be necessary depending on your setup. If you have all your files in e.g. `public/` and your server root points to `public` as well, you will want to use this option to get the correct relative url.

Lastly, the `runtimeCaching` is an array of objects to specify <a href="https://github.com/GoogleChrome/sw-precache#runtimecaching-arrayobject" class="o-link o-link--decorated">runtime caching options</a>. Each object must consist of a `urlPattern` which is either a string or a regex. And a handler which is either the name of one of the handlers supplied in the underlying <a href="https://googlechrome.github.io/sw-toolbox/api.html#handlers" class="o-link o-link--decorated">sw-toolbox package</a> or a custom handler function. We are using `cacheFirst` which will try to get something from the cache first, and only if it is not available, check the network.

Our first `urlPattern` matches any local resource we load, so everything will be cached the first time it is requested.
The second patter matches any request loaded via `googleapis.com` which I use for web fonts. So those will be cached as well.

#### callback
Lastly we call `done` as our callback, so the `gulp task` knows, that it is finished and we can move on.

When you run `gulp service-worker` you should get a nicely created service worker at `public/service-worker.js` which includes the urls from your `urlsToPrefetch` array. Let's move on and add this task to our workflow.

### Stale assets & html
But what if I change my html and it is already cached by the runtime cache, you might ask. Don't fear, `sw-precache` has got your back here. By default it will automatically cache-bust all urls if the service worker changed, so your files are always up-to-date. However, this also means our revisioned files will be re-fetched. There is the `dontCacheBustUrlsMatching` option to address this issue, but we will do this in a later part.

### Patching it all together
In your gulp `default` task, add the `service-worker` task after your html task, by this point all your assets should be compiled, revisioned and your html files should be created. Now whenever you run `gulp` your service worker will be updated. Additionally you should add the `service-worker` task to the appropriate watch tasks. Whenever an asset that is prefetch or will be cached is changed, you need to update your service worker as well.

If you did not do it already, don't forget to load the `service-worker-registration.js`.

Wow, we are done. It wasn't to hard now, was it? If you have any issues, let me know on <a class="o-link o-link--decorated" title="Reply to my article with a tweet" href="https://twitter.com/intent/tweet?text=Making+your+website+fast+%26+offline+available+with+a+service+worker https://veare.blog.dev/blog/170326-building-a-pwa-from-your-existing-website-part-2%20via%20%40lukasoppermann&amp;source=webclient">Twitter</a>, maybe I can help you.
