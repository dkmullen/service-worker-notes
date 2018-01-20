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
