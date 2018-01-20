## Service Worker notes

A service worker is a bit of JS that doesn't interact with the DOM, but stands between the app and the web. HTTP requests go thru it and can be redirected to a cache first, for ex.

Example: This shows the power of an SW as it logs EVERY req the page makes, which can then be manipulated. So SWs run on https only (localhost being an exception)

```
self.addEventListener('fetch', function(event) {
  console.log(event.request);
});
```

Register it like:

```
navigator.serviceWorker.register('/sw.js', { scope: '/my-app/'} ).then((reg) => {
  console.log('Success');
}).catch((err) => {
  console.log(err);
});
```

Providing the scope causes the SW to control any page from the given url deeper. Note that the above will control `/my-app/` but not `/my-app` (w/o the trailing slash)

Often, you don't have to set scope; just put it in the right place. Default scope is the path the SW sits in. `/foo/bar/sw/js` = a scope of `/foo/bar/`

To allow for browsers that don't support SW, wrap the above in:

```
if (navigator.serviceWorker) {

}
```
Or, just one line: `if(!navigator.serviceWorker) return;`

JS doesn't have private methods. Common practice to start methods with an underscore if they will only be called by other methods of this object. `this._openSocket()`

##Lifecycle

SW takes control of a page when it is loaded, so any SW put in place after page-load has to wait for a new instance of the page ie, a refresh.

If the SW changes, a new SW is created and put into waiting, and will not take effect till all pages using the current version are gone. This ensures only one version of your site is running at a time. REFRESH doesn't overcome this because there is a link between one version of a page and a refreshed version, so the existing SW never stops controlling the page. The new one won't take over till the page is closed or you navigate to another page. For this reason, set SW cache times low, even zero.

## Service Worker notes 2

### Dev tools
- In Chrome **Console** tab, can **change scope** from default (top) to sw.js.
- In Chrome **Sources** tab, can open the SW and use JS debugging to step thru it line-by-line;
- In Chrome **Application** tab, find SW and checkbox to **update on reload**. This is also where you will see **SWs in waiting**. BTW `SHFT-Refresh` doesn't seem to work anymore (to update SW on reload)???
- In Chrome **Network** tab, find response headers, etc.


### What can we DO?
Here we intercept all requests and respond with custom text (and set the header to HTML):
```
self.addEventListener('fetch', function(event) {
  event.respondWith(
    new Response('<div class="a-winner-is-me">Heel <strong>it!</strong></div>', {
      headers : {'Content-Type' : 'text/html; charset=utf-8'}
    })
  );
});
```
`event.respondWith()` takes either a response or a promise that resolves to a response.
`fetch()` returns a promise that resolves to a response. `fetch()` performs a normal browser fetch so results may come from cache or network.

Here we intercept only reqs that end with .jpg and respond to those with our own git. All other reqs go through normally.
```
self.addEventListener('fetch', function(event) {
  if (event.request.url.endsWith('.jpg')) {
    event.respondWith(
      fetch('/imgs/dr-evil.gif')
    );
  }
});
```
`endsWith()` - An incredibly handy new-to-me string method!

In the following, we respond with a network fetch for the request, just as the browser would do if we weren't inserting this service worker. But here, we intercept the response coming back and if we get a 404 or if the request fails (as when the network is down), we insert our own error messages. Otherwise, if no 404 and no fail, we simply `return response`, ie pass along the standard response.

```
self.addEventListener('fetch', function(event) {
  event.respondWith(
    fetch(event.request)
      .then((response) => {
        if(response.status == 404) {
          return new Response('Whoops, not found!');
          // could also return a fetch, ie - return fetch('/imgs/dr-evil.gif')
        }
        return response;
      })
      .catch(() => {
        return new Response('Uh oh, that failed.');
      })
  );
});
```

## Service Worker notes 3

### The cache API
In dev tools, find your cache in **Application** tab, then **Cache Storage**

To create or open a cache, use open.
```
caches.open('my-new-cache').then ...
```

The 'cache-box' contains req-res pairs from any secure origin. Can store anything there, either from our own origin or elsewhere on the web.

Add to a cache with:
`cache.put(req, res)` or `cache.addAll(['/foo', '/bar'])` which takes an array of reqs or urls, fetches them, puts the req-res pairs into the cache. Note that with addAll, if one req fails, all fail, and addAll uses fetch() under the hood...

To retrieve from a cache:
`cache.match(req)` or `caches.match(req)` w/c tries to find a match in ANY cache, starting with the oldest.

To delete: `caches.delete(cacheName)`
To get the names of all caches: `caches.keys()`


## The install event
When a page encounters an SW, it runs an install event, and the SW can't be used until it finishes. We can use install to load up a cache.

In the following example, we add an event listener to detect install, and when it happens, the event it will return is told to waitUntil the urls are loaded into a cache.
```
self.addEventListener('install', function(event) {
  var urlsToCache = [
    '/',
    'js/main.js',
    'css/main.css',
    'imgs/icon.png',
    'https://fonts.gstatic.com/s/roboto/v15/2UX7WLTfW3W8TclTUvlFyQ.woff',
    'https://fonts.gstatic.com/s/roboto/v15/d-6IYplOFocCacKzxwXSOD8E0i7KZn-EPnyo3HZu7kw.woff'
  ];

  event.waitUntil(
    // TODO: open a cache named 'wittr-static-v1'
    // Add cache the urls from urlsToCache
    caches.open('wittr-static-v1')
      .then((cache) => {
        return cache.addAll(urlsToCache);
      })
  );
});
// This last bit listens for fetch, then looks in the cache for any items there.
// It returns cache items if they exist, returns a network fetch if not
self.addEventListener('fetch', function(event) {
  event.respondWith(
    caches.match(event.request)
      .then((response) => {
        if (response) return response;
        return fetch(event.request);
      })
  );
});
```

## The activate event
What if we change CSS, for example? We have cached the old CSS, and it won't update. Caches get initialized when a new SW is installed, but there is no new SW after something like a CSS change. So how can we work with the SW to pick up changes?

To force an update, we need to make a change to SW. That makes it a new SW and it will then get its own install event (in which it will load the new cache) and be put into waiting. BTW, it won't automatically put items in a new cache. We have to specify that with a new cache name, and we should because we don't want to disrupt pages still using the old cache.

We can spin up a new SW just by making a minor change, like renaming the cache. What we haven't seen yet is how to get rid of the old cache. That's where activate comes in. Activate fires when a new SW goes online, and this is a good time to get rid of old caches.

```
self.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.delete(cacheName);
  );
});
```
