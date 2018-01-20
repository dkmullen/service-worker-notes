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
