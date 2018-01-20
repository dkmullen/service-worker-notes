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
