# Awesome JavaScript Snippets

## Animation

```js
// Double RAF is useful for ensuring that animations start before expensive rendering is done.
// It helps provide smoother user experience by making animations feel reactive.
// Normal rendering would block the animation from starting.
// https://github.com/UseWebPlatform/awesome-web-platform/js-snippets.md#animation
// ES6
const doubleRAF = cb => requestAnimationFrame(() => requestAnimationFrame(cb));

// ES7
const requestAnimationFramePromise = () => new Promise(resolve => requestAnimationFrame(resolve));
const doubleRAF = async () => { 
  await requestAnimationFramePromise();
  await requestAnimationFramePromise();
};

// Examples
// With double RAF as shown here the rendering function safely runs in the main thread
// after the animation has already started.
// ES6
button.addEventListener('click', function() {
  element.classList.add('animating');
  doubleRAF(renderNextView);
});

// ES7
element.classList.add('animating');
await doubleRAF();
renderNextView();
```
[source](https://stackoverflow.com/questions/44145740/how-does-double-requestanimationframe-work),
[source2](https://youtu.be/mmq-KVeO-uU?t=791),
[source3](https://github.com/surma/lurkk.it/blob/fe56dbb641127b6fc5ac5acec114f52f465e55a0/src/utils/animation.ts#L15)

## Document

```js
// Clear a focus.
// https://github.com/UseWebPlatform/awesome-web-platform/js-snippets.md#document
document.activeElement.blur();
```
[source](https://stackoverflow.com/a/2520670/1614237)

## Network

```js
// Don't run if the user is on 2G or if Save-Data is enabled.
// https://github.com/UseWebPlatform/awesome-web-platform/js-snippets.md#network
if (conn = navigator.connection) {
  if ((conn.effectiveType || '').includes('2g') || conn.saveData) return;
}
```
[source](https://github.com/GoogleChromeLabs/quicklink/blob/master/src/prefetch.mjs#L104)

## Optimization

```js
// Defer another network requests after initial frame is rendered to keep the page load performant.
// https://github.com/UseWebPlatform/awesome-web-platform/js-snippets.md#optimization
const defer = fn => {
  const raf = window.requestAnimationFrame || window.mozRequestAnimationFrame ||
              window.webkitRequestAnimationFrame || window.msRequestAnimationFrame;
  if (raf) raf(() => window.setTimeout(fn, 0));
  else window.addEventListener('load', fn);
}

const deferScript = scriptUrl => {
  defer(() => {
    const script = document.createElement('script');
    script.async = true;
    script.src = scriptUrl;
    document.body.appendChild(script);
  });
}

const deferStyles = noscriptId => {
  defer(() => {
    const addStylesNode = document.getElementById(noscriptId);
    const replacement = document.createElement('div');
    replacement.innerHTML = addStylesNode.textContent;
    document.body.appendChild(replacement);
    addStylesNode.parentElement.removeChild(addStylesNode);
  });
}

// Examples
deferScript('https://www.google-analytics.com/analytics.js');

// Check that service workers are registered.
if ('serviceWorker' in navigator) {
  defer(() => navigator.serviceWorker.register('/sw.js'));
}

<noscript id="deferred-styles">
  <link rel="stylesheet" href="https://fonts.googleapis.com/css?family=Roboto:500&subset=latin-ext">
</noscript>
<script>
  deferStyles('deferred-styles');
</script>
```
[source](https://developers.google.com/speed/docs/insights/OptimizeCSSDelivery)

## Performance

```js
// Polyfill/shim for the requestIdleCallback and cancelIdleCallback API.
// https://github.com/UseWebPlatform/awesome-web-platform/js-snippets.md#performance
window.requestIdleCallback = window.requestIdleCallback ||
  cb => {
    const start = Date.now();
    return setTimeout(() => {
      cb({
        didTimeout: false,
        timeRemaining: () => Math.max(0, 50 - (Date.now() - start)),
      });
    }, 1);
  };

window.cancelIdleCallback = window.cancelIdleCallback || id => clearTimeout(id);
```
[source](https://developers.google.com/web/updates/2015/08/using-requestidlecallback#checking_for_requestidlecallback)

```js
// Get anytime a single task blocks the main thread for more than 50ms.
// https://github.com/UseWebPlatform/awesome-web-platform/js-snippets.md#performance
function sendLongTaskDataToGoogleAnalytics(entryList) {
  // Assumes the availability of requestIdleCallback (or a shim).
  requestIdleCallback(() => {
    for (const entry of entryList.getEntries()) {
      ga('send', 'event', {
        eventCategory: 'Performance Metrics',
        eventAction: 'longtask',
        eventValue: Math.round(entry.duration),
        eventLabel: JSON.stringify(entry.attribution),
      });
    }
  });
}

// Create a PerformanceObserver and start observing Long Tasks.
new PerformanceObserver(sendLongTaskDataToGoogleAnalytics).observe({
  entryTypes: ['longtask'],
});
```
[source](https://philipwalton.com/articles/why-web-developers-need-to-care-about-interactivity/)

```js
// Using requestIdleCallback for sending analytics data.
// https://github.com/UseWebPlatform/awesome-web-platform/js-snippets.md#performance
const processPendingAnalyticsEvents = deadline => {
  // Reset the boolean so future rICs can be set.
  isRequestIdleCallbackScheduled = false;

  // If there is no deadline, just run as long as necessary.
  // This will be the case if requestIdleCallback doesn’t exist.
  if (typeof deadline === 'undefined')
    deadline = { timeRemaining: () => Number.MAX_VALUE };

  // Go for as long as there is time remaining and work to do.
  while (deadline.timeRemaining() > 0 && eventsToSend.length > 0) {
    var evt = eventsToSend.pop();
    ga('send', 'event',
        evt.category,
        evt.action,
        evt.label,
        evt.value);
  }

  // Check if there are more events still to send.
  if (eventsToSend.length > 0) schedulePendingEvents();
}

const schedulePendingEvents = () => {
  // Only schedule the rIC if one has not already been set.
  if (isRequestIdleCallbackScheduled) return;

  isRequestIdleCallbackScheduled = true;

  // Assumes the availability of requestIdleCallback (or a shim).
  // Wait at most two seconds before processing events.
  requestIdleCallback(processPendingAnalyticsEvents, { timeout: 2000 });
}

let eventsToSend = [];
let isRequestIdleCallbackScheduled = false;

// Example
function onNavOpenClick () {
  // Animate the menu.
  menu.classList.add('open');

  // Store the event for later.
  eventsToSend.push(
    {
      category: 'button',
      action: 'click',
      label: 'nav',
      value: 'open'
    });

  schedulePendingEvents();
}
```
[source](https://developers.google.com/web/updates/2015/08/using-requestidlecallback#using_requestidlecallback_for_sending_analytics_data)

## Time

```js
// Fastest method to get timestamp.
// https://github.com/UseWebPlatform/awesome-web-platform/js-snippets.md#time
const timestamp = Date.now();
```
[source](https://stackoverflow.com/a/51067600/1614237)

```js
// Timestamp in milliseconds.
// https://github.com/UseWebPlatform/awesome-web-platform/js-snippets.md#time
const timestampInMs = window.performance && window.performance.now && window.performance.timing &&
                      window.performance.timing.navigationStart ?
                      window.performance.now() + window.performance.timing.navigationStart : Date.now();
```
[source](https://stackoverflow.com/a/221297/1614237)

## URL

```js
// Remove URL parameters without refreshing page.
// https://github.com/UseWebPlatform/awesome-web-platform/js-snippets.md#url
window.history.replaceState(null, null, window.location.pathname);
```        
[source](https://stackoverflow.com/a/41061471/1614237)

## Another snippets

- [30 seconds of code](https://github.com/30-seconds/30-seconds-of-code)
- [Code to go](https://codetogo.io)
- [CSS Tricks JS Snippets](https://css-tricks.com/snippets/javascript/)
- [Hyper JavaScript Snippets for VS Code](https://marketplace.visualstudio.com/items?itemName=t7yang.hyper-javascript-snippets)
- [JavaScript (ES6) code snippets for VS Code](https://marketplace.visualstudio.com/items?itemName=xabikos.JavaScriptSnippets)
- [W3Schools How To](https://www.w3schools.com/howto/default.asp)
