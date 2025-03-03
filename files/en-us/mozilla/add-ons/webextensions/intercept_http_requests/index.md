---
title: Intercept HTTP requests
slug: Mozilla/Add-ons/WebExtensions/Intercept_HTTP_requests
tags:
  - Add-ons
  - Extensions
  - How-to
  - WebExtensions
---
{{AddonSidebar}}

To intercept HTTP requests, use the {{WebExtAPIRef("webRequest")}} API.
This API enables you to add listeners for various stages of making an HTTP request.
In the listeners, you can:

- get access to request headers and bodies, and response headers
- cancel and redirect requests
- modify request and response headers

In this article we'll look at three different uses for the `webRequest` module:

- Logging request URLs as they are made.
- Redirecting requests.
- Modifying request headers.

## Logging request URLs

Create a new directory called "requests".
In that directory, create a file called "manifest.json" which has the following contents:

```json
{
  "description": "Demonstrating webRequests",
  "manifest_version": 2,
  "name": "webRequest-demo",
  "version": "1.0",

  "permissions": [
    "webRequest",
    "<all_urls>"
  ],

  "background": {
    "scripts": ["background.js"]
  }
}
```

Next, create a file called "background.js" with the following contents:

```js
function logURL(requestDetails) {
  console.log("Loading: " + requestDetails.url);
}

browser.webRequest.onBeforeRequest.addListener(
  logURL,
  {urls: ["<all_urls>"]}
);
```

Here we use {{WebExtAPIRef("webRequest.onBeforeRequest", "onBeforeRequest")}} to call the `logURL()` function just before starting the request. The `logURL()` function grabs the URL of the request from the event object and logs it to the browser console.
The `{urls: ["<all_urls>"]}` [pattern](/en-US/docs/Mozilla/Add-ons/WebExtensions/Match_patterns) means we will intercept HTTP requests to all URLs.

To test it out:
- [Install the extension](https://extensionworkshop.com/documentation/develop/temporary-installation-in-firefox/)
- Open the [Browser Console](https://firefox-source-docs.mozilla.org/devtools-user/browser_console/) (use <kbd>Ctrl + Shift + J</kbd>)
- Enable *Show Content Messages* in the menu:

  ![Browser console menu : Show Content Messages](browser_console_show_content_messages.png)
- Open some Web pages. 

In the Browser Console, you should see the URLs for any resources that the browser requests.
For example, this screenshot shows the URLs from loading a Wikipedia page:

![Browser console menu : URLs from extension](browser_console_url_from_extension.png)

<!-- {{EmbedYouTube("X3rMgkRkB1Q")}} -->

## Redirecting requests

Now let's use `webRequest` to redirect HTTP requests. First, replace manifest.json with this:

```json
{

  "description": "Demonstrating webRequests",
  "manifest_version": 2,
  "name": "webRequest-demo",
  "version": "1.0",

  "permissions": [
    "webRequest",
    "webRequestBlocking",
    "https://developer.mozilla.org/"
  ],

  "background": {
    "scripts": ["background.js"]
  }

}
```

The changes here are to:

- add the `webRequestBlocking` [`permission`](/en-US/docs/Mozilla/Add-ons/WebExtensions/manifest.json/permissions).
  This extra permission is needed when an extension wants to modify a request.
- replace the `<all_urls>` permission with individual [host permissions](/en-US/docs/Mozilla/Add-ons/WebExtensions/manifest.json/permissions#host_permissions), as this is good practice to minimize the number of requested permissions.

Next, replace **background.js** with this:

```js
var pattern = "https://developer.mozilla.org/*";
var targetUrl = "https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/Your_second_WebExtension/frog.jpg";

function redirect(requestDetails) {
  console.log("Redirecting: " + requestDetails.url);
  if (requestDetails.url === targetUrl) {
    return;
  }
  return {
    redirectUrl: targetUrl
  };
}

browser.webRequest.onBeforeRequest.addListener(
  redirect,
  {urls:[pattern], types:["image"]},
  ["blocking"]
);
```

Again, we use the {{WebExtAPIRef("webRequest.onBeforeRequest", "onBeforeRequest")}} event listener to run a function just before each request is made.
This function replaces the `redirectUrl` with the target URL specified in the function. In this case, the frog image from the [your second extension tutorial](/en-US/docs/Mozilla/Add-ons/WebExtensions/Your_second_WebExtension).

This time we are not intercepting every request: the `{urls:[pattern], types:["image"]}` option specifies that we should only intercept requests (1) to URLs residing under "https\://developer.mozilla.org/" (2) for image resources.
See {{WebExtAPIRef("webRequest.RequestFilter")}} for more on this.

Also note that we're passing an option called `"blocking"`: we need to pass this whenever we want to modify the request.
It makes the listener function block the network request, so the browser waits for the listener to return before continuing.
See the {{WebExtAPIRef("webRequest.onBeforeRequest")}} documentation for more on `"blocking"`.

To test it out, open a page on MDN that contains a lot of images (for example [the page listing extension user interface components](/en-US/docs/Mozilla/Add-ons/WebExtensions/user_interface)), [reload the extension](https://extensionworkshop.com/documentation/develop/temporary-installation-in-firefox/#reloading_a_temporary_add-on), and then reload the MDN page. You will see something like this:

![Images on a page replaced with a frog image](beastify_by_redirect.png)

## Modifying request headers

Finally we'll use `webRequest` to modify request headers.
In this example we'll modify the "User-Agent" header so the browser identifies itself as Opera 12.16, but only when visiting pages under http\://useragentstring.com/".

Update your **manifest.json** to include `http://useragentstring.com/`

```json
{
  "description": "Demonstrating webRequests",
  "manifest_version": 2,
  "name": "webRequest-demo",
  "version": "1.0",

  "permissions": [
    "webRequest",
    "webRequestBlocking",
    "http://useragentstring.com/"
  ],

  "background": {
    "scripts": ["background.js"]
  }
}
```

Replace "background.js" with code like this:

```js
var targetPage = "http://useragentstring.com/*";

var ua = "Opera/9.80 (X11; Linux i686; Ubuntu/14.10) Presto/2.12.388 Version/12.16";

function rewriteUserAgentHeader(e) {
  e.requestHeaders.forEach(function(header){
    if (header.name.toLowerCase() == "user-agent") {
      header.value = ua;
    }
  });
  return {requestHeaders: e.requestHeaders};
}

browser.webRequest.onBeforeSendHeaders.addListener(
  rewriteUserAgentHeader,
  {urls: [targetPage]},
  ["blocking", "requestHeaders"]
);
```

Here we use the {{WebExtAPIRef("webRequest.onBeforeSendHeaders", "onBeforeSendHeaders")}} event listener to run a function just before the request headers are sent.

The listener function will be called only for requests to URLs matching the `targetPage` [pattern](/en-US/docs/Mozilla/Add-ons/WebExtensions/Match_patterns).
Also note that we've again passed `"blocking"` as an option. We've also passed `"requestHeaders"`, which means that the listener will be passed an array containing the request headers that we expect to send.
See {{WebExtAPIRef("webRequest.onBeforeSendHeaders")}} for more information on these options.

The listener function looks for the "User-Agent" header in the array of request headers, replaces its value with the value of the `ua` variable, and returns the modified array.
This modified array will now be sent to the server.

To test it out, open [useragentstring.com](http://useragentstring.com/) and check that it identifies the browser as Firefox.
Then reload the extension, reload [useragentstring.com](http://useragentstring.com/), and see that Firefox is now identified as Opera.

![useragentstring.com showing details of the modified user agent string](modified_request_header.png)

## Learn more

To learn about all the things you can do with the `webRequest` API, see its [reference documentation](/en-US/docs/Mozilla/Add-ons/WebExtensions/API/webRequest).
