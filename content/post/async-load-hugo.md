---
title: "Asynchronously loading pages in a Hugo site"
date: 2017-08-30T23:39:16-07:00
---

I wrote a Hugo "theme" that loads its pages asynchronously. Play with it
[here](https://aarmea.github.io/mfw-singlepage/) or read the code
[here](https://github.com/aarmea/mfw-singlepage/tree/e17c04d/).

{{< rawhtml >}}
  <video autoplay loop muted class="playpause-with-visibility">
    <source src="/post/async-load-hugo/main.webm" type="video/webm">
    <source src="/post/async-load-hugo/main.mp4" type="video/mp4">
  </video>
{{< /rawhtml >}}

<!--more-->

# Using Hugo for a podcast site

A few weeks ago, one of my friends asked me to help with the website for a
podcast he was starting. Two of our requirements were that it needed to be a
static site, mostly to save on hosting costs, and that it must have seamless
playback: its audio player shouldn't ever be interrupted by navigating through
the site. The websites of podcast [99%
Invisible](http://99percentinvisible.org/) and audio hosting service
[SoundCloud](https://soundcloud.com/) both have excellent implementations of
seamless playback.

I eventually settled on the static site generator [Hugo](https://gohugo.io/)
because it [natively supports post categories independently from content
types](https://gohugo.io/content-management/types/), which we need to post both
podcast episodes and articles but allow them to appear together in category
listings. [Pagify.js](https://github.com/cmPolis/Pagify) seemed promising for
seamless playback (and it even has a [Jekyll
plugin](https://github.com/JCB-K/singlepage-jekyll)), but because it only loads
the actual content of a page via an AJAX call, that content will not be
available to search engines or other users that disable JavaScript.

My prototype implements seamless playback in Hugo from scratch. Once you play
the podcast, click around in the site. This will not stop playback -- internal
links load asynchronously and external links load in a new tab. With JavaScript
disabled, you will at least be able to read the text content of the site.

# Pardon the dust

The code here is the *absolute minimum* needed to implement this functionality,
so I'm intentionally ignoring some best practices: there's no CSS at all and the
JavaScript is defined inline, among others.

# Writing a Hugo theme

[Hugo's documentation on creating a theme](https://gohugo.io/themes/creating/)
tells you to run `hugo new theme [name]`, which is supposed to generate a
scaffold of your new theme. I needed a place to define a base container template
that Hugo would wrap around the site's content. It turns out that [the correct
file for this is
`layouts/_default/baseof.html`](https://gohugo.io/templates/base/) but as of
this post, [Hugo does not create this file for
you](https://github.com/gohugoio/hugo/issues/3576).

Inside this file, I wrote a minimal HTML page with a header, a footer, and room
for content. Hugo will populate `{{ block "main" . }}` with the page's content
when the site is generated, and our JavaScript will change `#mainContent` as the
user clicks on links.

```html
<!DOCTYPE html>
<html lang="{{ .Site.LanguageCode }}">
  <head>
    {{ .Hugo.Generator }}
    <meta charset="utf-8">
    <title>{{ block "title" . }}{{ .Title }} | {{ .Site.Title }}{{ end }}</title>
  </head>
  <body>
    <header>
      <h1><a href="{{ "/" | relURL }}">{{ .Site.Title }}</a></h1>
    </header>
    <main id="mainContent">{{ block "main" . }}{{ end }}</main>
    <footer>
      <a href="https://github.com/aarmea/mfw-singlepage">mfw-singlepage</a>
      theme for <a href="https://gohugo.io">Hugo</a>
    </footer>
  </body>
</html>
```

The `<header>` and `<footer>` should probably be split out into their own
[partial templates](https://gohugo.io/templates/partials/) so that you can
cleanly add your own embedded media here. However, they should not vary per page
because they are never reloaded to avoid interrupting media playback.

While `baseof.html` defines the base template, the actual pages are defined
elsewhere and work by defining [variables](https://gohugo.io/variables/), like
the previously mentioned `{{ block "main" . }}`, that are substituted into the
template. The homepage layout is at
[`layouts/index.html`](https://github.com/aarmea/mfw-singlepage/blob/e17c04dbc59b93b6dbb50b0cfa8660fb464d864d/layouts/index.html)
and the individual post layout is at
[`layouts/_default/single.html`](https://github.com/aarmea/mfw-singlepage/tree/e17c04dbc59b93b6dbb50b0cfa8660fb464d864d/layouts/_default).
Default layouts can be overridden on a per-section basis in `layouts/[section]`.

When writing the theme, I also referenced the source of the [After
Dark](https://comfusion.github.io/after-dark/) and
[hugo-xmin](https://github.com/yihui/hugo-xmin) themes as examples.

# Loading pages asynchronously

Loading the site's pages is done in a few steps: override the link action,
retrieve and display the content, and update the back stack and displayed URL.

## Override the link click action

The easiest way to do this is to register a click handler on the `<body>` and do
nothing if the click event was not on a link:

```javascript
window.onload = function() {
  // Make links load asynchronously
  document.body.addEventListener("click", function(event) {
    if (event.target.tagName !== "A")
      return;

    // History API needed to make sure back and forward still work
    if (history === null)
      return;

    event.preventDefault();

    // External links should instead open in a new tab
    var newUrl = event.target.href;
    var domain = window.location.origin;
    if (typeof domain !== "string" || newUrl.search(domain) !== 0) {
      window.open(newUrl, "_blank");
    } else {
      loadPage(newUrl);
      history.pushState(null /*stateObj*/, "" /*title*/, newUrl);
    }
  });
}
```

Setting up this behavior this way, as opposed to adding the click handler to
every `<a>` individually, means that I won't have to register more handlers
when loading a new page.

It might be better to use the
[DOMContentLoaded](https://developer.mozilla.org/en-US/docs/Web/Events/DOMContentLoaded)
event or [`$(document).ready` from
jQuery](https://learn.jquery.com/using-jquery-core/document-ready/) here because
this functionality doesn't depend on images and other content loading, but
[`window.onload` waits for those
anyway](http://mdn.beonex.com/en/DOM/window.onload.html). Alternately, I could
have just given the click handler a name and set it directly on the `<body
onclick="...">`.

## Retrieve and display the new content

`loadPage()` does the AJAX call needed to get the new content. Using a raw
[`XMLHttpRequest`](https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest)
isn't exactly a one-liner like [jQuery's
`get()`](https://api.jquery.com/jQuery.get/), but it's not too far off:

```javascript
function loadPage(newUrl) {
  var httpRequest = new XMLHttpRequest();
  httpRequest.onreadystatechange = function() {
    if (httpRequest.readyState !== XMLHttpRequest.DONE)
      return;

    var newDocument = httpRequest.responseXML;
    if (newDocument === null)
      return;

    var newContent = httpRequest.responseXML.getElementById("mainContent");
    if (newContent === null)
      return;

    document.title = newDocument.title;

    var contentElement = document.getElementById("mainContent");
    contentElement.replaceWith(newContent);
  }

  httpRequest.responseType = "document";
  httpRequest.open("GET", newUrl);
  httpRequest.send();
};
```

`XMLHttpRequest` is capable of parsing the response as HTML. To do this, set its
`responseType` to `"document"` before sending the request and wait for
`readyState` to become `XMLHttpRequest.DONE`. Then `responseXML` will contain
the root element of the response and methods like `getElementById` will work on
it as expected.

After getting the `#mainContent` element in both the visible
document and the newly loaded one, switching them out is just a call to
`replaceWith`.

If Internet Explorer support is not a concern to you, you could consider using
[Fetch](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API) instead of
XMLHttpRequest.

## Update the back stack

At this point, the site works and you can freely browse without interrupting
media playback, but the back and forward buttons don't do what you'd expect them
to. The [History
API](https://developer.mozilla.org/en-US/docs/Web/API/History_API) allows
JavaScript to modify the back stack to enable the back and forward buttons to
work in a single page site.

First, check if the history API is available. I did this check in the click
handler before overriding its behavior:

```javascript
// History API needed to make sure back and forward still work
if (history === null)
  return;
```

Further down, the click handler also calls `pushState` after displaying the new
page to add it to the history and change the shown URL:

```javascript
history.pushState(null /*stateObj*/, "" /*title*/, newUrl);
```

Finally, the `onpopstate` handler is called when the back button is clicked.
When this happens, the page needs to reload the content from that URL.

```javascript
window.onpopstate = function(event) {
  loadPage(window.location);
}
```

I could probably save bandwidth that will be wasted when the same page is loaded
more than once by caching the content.

# Saying no to jQuery

I originally avoided jQuery for this prototype as a learning exercise, but in
hindsight using it would not have helped much.

In my case, jQuery could have helped by providing a better place to register the
click handler, simplifying the AJAX call, and slightly shortening the call to
locate the target element. Working around these isn't that annoying and isn't
worth the 32 KB of bandwidth it takes to load jQuery.

Maybe your new site doesn't need jQuery either. See [You Might Not Need
jQuery](http://youmightnotneedjquery.com/) and [You Don't Need
jQuery](https://blog.garstasio.com/you-dont-need-jquery/why-not/) for more.
