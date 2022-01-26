---
title: Django, HTMX and Alpine
description: The old is new again - a better way to build web apps?
date: 2022-01-24
tags:
  - django
  - javascript
  - python
  - web development
layout: layouts/post.njk
---

![The Winter Panorama, Champéry, Switzerland](/img/joel-jasmin-forestbird-Zi9yS95gDUQ-unsplash.jpg)

Photo by [Joel & Jasmin Førestbird](https://unsplash.com/@theforestbirds?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/alps?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)
Over the past year or so I've been closely following an interesting trend in web development in which a wave of new libraries and frameworks allow developers to build modern web applications more simply, cheaply and quickly.

The current dominant paradigm in web development is the Single Page Application or SPA architecture. Typically this consists of a backend API or APIs connected to a frontend built in a Javascript framework such as React, Vue or Svelte. The API communicates with the frontend through JSON payloads, with the frontend having sole responsiblity for rendering the data in the DOM.

This model has some advantages. It allows separation of concerns: if we wanted to serve not just web pages but, say, an Android app, the Android app could talk to the same API as our web application. The backend logic can be changed completely - you could decide one day to migrate from Django to Rails, say, and as long as your API stays the same the frontend doesn't need to know or care (the same, of course, applies to the frontend). This enables developers in different teams and disciplines - backend and frontend specialists - to work independently. Furthermore, a frontend framework like React can - at least in theory - provide a smoother experience to the end-user, without full page loads and slow and janky form processing experiences. It's easy to see why the SPA model has become the de-facto pattern of web development in the past decade.

The SPA model however also comes with a lot of downsides. Logic such as form validation has to be duplicated between client and server. You may have to host two separate applications in different domains, adding complexity to otherwise "solved problems" such as authentication. A bug in your Javascript code may render not merely a semi-functional site, but a blank page with no clue for the developer or user on how to fix. SEO is more problematic as Javascript-rendered sites are opaque (or at least suboptimal) to search engine crawlers and social media sharing services. A web page might initially load more quickly, but the user is left looking at numerous spinning gifs and blinking wireframes while half a dozen API calls load individual parts of the page. There are solutions to these issues, from SSR to GraphQL to [CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS), but they entail further complexity and more dependencies.

All of this additional complexity translates to a higher cost in time and money to get your product to market or add new features, and a higher cost in salary or consultant fees as you have to hire larger teams of specialists (and managing larger, multi-disiplinary teams brings with it additional management and communication overhead). It also creates a larger surface area for bugs and potential security issues (the sheer number of NPM modules required to run an average SPA frontend alone means that sooner or later, [you are going to inadvertently import insecure or even malicious code](https://arstechnica.com/information-technology/2021/09/npm-package-with-3-million-weekly-downloads-had-a-severe-vulnerability/), no matter how much auditing you do). Moreover these dependencies amount to huge initial loads running into the megabytes even after asset compression - perhaps irrelevant to developers working with expensive, high-end desktops in urban locations with excellent high-speed internet, but not so much fun to someone using an older mobile device in a rural area with poor connectivity. A sufficiently skilled team can of course mitigate all of these problems; but [most teams do not have sufficient skill (or the available time and resources) to build a such a high quality SPA](https://www.baldurbjarnason.com/2021/single-page-app-morality-play).

However, we have to acknowledge that SPAs, no matter their costs and disadvantages, became popular because they solved real problems with the old server-side rendering model, for example full page reloads on every button or link click or the tiresome post-validate-repost cycle of form validation, and provided a better means of delivering more complex and interactive applications.

While the popularity of SPA frameworks grew over the past decade, however, others began to work on an alternative approach to building sophisticated web applications.

To mitigate the problems inherent with the traditional server-rendered architecture, a common solution before the current frontend era was to use AJAX to fetch discrete chunks of HTML from the server and insert them into the DOM, rather than do a full page load on every request. Very often this made use of [jQuery.load()](https://api.jquery.com/load/), given the popularity of jQuery in this period. This was fine for isolated cases, but became difficult to develop and maintain in larger and more complex apps. More advanced frameworks such as Angular and React provided more structure and optimizations for handling complex DOM manipulations in large frontend-heavy projects.

This pattern - sometimes disparagingly referred to as FAJAX, i.e. "Fake AJAX" - didn't die out completely during the Javascript framework boom of the 2010s. A number of libraries were released which provided easier and more structured means of handling AJAX and DOM interaction, including:

* [PJAX](https://github.com/defunkt/jquery-pjax)
* [Turbolinks](https://github.com/turbolinks/turbolinks)
* [Intercooler](https://github.com/bigskysoftware/intercooler-js)
* [Unpoly](https://unpoly.com/)
* [Laravel Livewire](https://laravel-livewire.com/)
* [Phoenix Channels](https://www.phoenixframework.org/)

Turbolinks was subsumed into the [Hotwire](https://hotwire.dev) project, developed by the Rails team. A successor to Intercooler, also developed by Carson Gross, was released in 2021: [HTMX](https://htmx.org).

## HTMX

HTMX makes it easy to progressively add AJAX functionality to a traditional server-side rendered web application through a set of custom HTML attributes. For example, suppose we have a "Subscribe" button on a web site, so you can follow updates from another user or content channel. When this button is clicked,  the user is subscribed to the channel and the button text should be updated to "Unsubscribe".

A traditional server-rendered application might look like this:

``` html
  <form method="POST" action="/subscribe/12345/">
    <button type="submit">Subscribe</button>
  </form>
```

Clicking the button submits our form to the server in a POST action. The server code would likely have to respond with an HTTP redirect back to the same page, forcing a full page reload. A button in of itself can't do a POST, so we have to wrap it in a `<form>` tag.

A full page reload as well as being a janky user experience is inherently wasteful of server resources: for example, suppose you need to render a navbar, footer, page details and so forth, requiring additional database calls? Fine for an initial page load, but doing all that just to update a button text is a lot of wasted resources.

HTMX allows us to handle this in a more graceful fashion. Using custom attributes with an *hx-** prefix (HTML purists can also use *data-hx-*) the button itself can handle the POST:

```html
  <button hx-post="/subscribe/12345/"
          hx-target="this"
          hx-swap="outerHTML">Subscribe</button>
```

The `hx-post` attribute means "send an HTTP POST to this URL". `hx-target` provides the DOM to be swapped (it can be any valid DOM selector, such as an ID or class; `this` is a special designator meaning "this element"), and `hx-swap` the actual DOM manipulation to be done on the result - in this case, replace the entire `<button>` with whatever HTML is returned from the URL. These three attributes alone can do a lot of lifting; HTMX has a [couple dozen such directives](https://htmx.org/reference/) in its toolbox, providing all sorts of AJAX-related functionality without writing a single line of Javascript.

The URL endpoint - it doesn't matter what language it is, HTMX will work with any server language you like, from Python or PHP to Go or Rust - should then return an HTML snippet, something like this:

```html
  <button hx-post="/unsubscribe/12345/"
          hx-target="this"
          hx-swap="outerHTML">Unsubscribe</button>
```

As you can see the response changes our text from "Subscribe" to "Unsubscribe" and the HTTP POST url to the corresponding "unsubscribe" action. Other than doing whatever authentication and database updates are necessary to handle creating and removing subscriptions, we need only return a small HTML snippet rather than an entire web page. HTMX will then insert this new `<button>` into the DOM as directed.

HTMX itself is a small Javascript dependency. The easiest way to get started is to just use a CDN:

```html
  <script src="https://unpkg.com/htmx.org@1.6.1"
          integrity="sha384-tvG/2mnCFmGQzYC1Oh3qxQ7CkQ9kMzYjWZSNtrRZygHPDDqottzEJsqS4oUVodhW"
          crossorigin="anonymous"></script>
```

In addition to small AJAX interactions, you can use HTMX to handle full page navigation, using the `hx-boost` feature. This works in a similar way to Turbolinks/Hotwire: you return full pages from the server, but the differing DOM elements in the `<body>` are swapped rather than doing a full reload. This provides a smoother experience for the user when navigating around the site, similar to the experience of using an SPA. HTMX also provides support for "push" techniques such as [Server-Sent Events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events) (SSE) and WebSockets.

## HTMX and Django

Integration of HTMX with Django is quite straightforward and does not require anything more than including the CDN in your base template. One thing to bear in mind is that HTMX provides a number of [request headers](https://htmx.org/reference/#request_headers) so you can provide more efficient and targeted responses.

For example in a view you might wish to return an HTML snippet if you know the request originated from an HTMX action, or a full page response in other cases:

```python
  def my_view(request):
      if request.headers.get("HX-Request"):
          return TemplateResponse(request, "_some_snippet.html")
      else:
          return TemplateResponse(request, "full_page.html")
```

The `HX-Request` header is automatically added to all HTMX requests, so you can check that the request originated from an HTMX action.

You might also wish to add HTMX-specific headers to your outbound response. For example you can return an `HX-Redirect` header to instruct HTMX to do a client-side redirect to another location.

While it's not necessary for getting started with HTMX and Django, I strongly recommend using the [Django-HTMX package](https://github.com/adamchainz/django-htmx) developed by Adam Johnson. This provides a Django middleware and other useful helpers to make Django and HTMX integration more seamless. For example, if you install the django-htmx middleware the above code can be rewritten:


```python
  def my_view(request):
      if request.htmx:
          return TemplateResponse(request, "_some_snippet.html")
      else:
          return TemplateResponse(request, "full_page.html")
```

The middleware adds an `htmx` attribute to the `HTTPRequest` instance passed to all views.

## Alpine

Of course, Javascript can do a lot more than handle AJAX interactions. For example, you might have modals, drop-down menus, "flash" messages and other interactions and effects that are a bit more tricky to achieve with CSS.

A number of libraries have arisen to do this kind of in-page work, providing an easier way to organize your frontend code without requiring a large frontend framework or the "spaghetti" of vanilla JS or jQuery. One such solution, again part of the Hotwire family, is [Stimulus](https://stimulus.hotwired.dev/). Another framework, developed by Carson Gross and others is [Hyperscript](https://hyperscript.org/). My personal favourite however is [Alpine.js](https://alpinejs.dev/).

Alpine, like HTMX, uses attributes to enhance functionality. Typically these begin with *x-*. For example:

```html
  <button x-on:click="alert('hello')">Click me!</button>
```

The `x-on` directive can be shorted to just `@`:

```html
  <button @click="alert('hello')">Click me!</button>
```

Events can be modified further with special keywords: for example `@click.prevent` will prevent event propagation, and `@click.window` will trigger the event if any part of the page is clicked.

Alpine allows DOM manipulation using this declarative syntax - if you have experience with Vue, you'll see some similarities. For example, suppose you want to hide a button when it's clicked:

```html
  <button x-data="{show: true}"
          x-show="show"
          @click="show=false">Click me and I'll hide!</button>
```

The `x-data` attribute initializes data (as a Javascript object), scoped to that element (and any child elements). Here we set a default value for `show` to **true**. The `x-show` directive determines the condition that the element will appear i.e. as long as `show` is **true**. Finally, our `@click` event sets `show` to **false**, in which case the button will disappear.

You can also do class manipulation, again in a similar way to Vue:

```html
  <button x-data="{clicked: false}"
          :class="{'text-red': clicked}"
          @click="clicked=true">Click me!</button>
```

Thus when the `@click` event is triggered, the class `text-red` is applied to the `<button>` element.

Again as with HTMX, you can install Alpine.js using a CDN:

```html
  <script src="https://unpkg.com/alpinejs@3.7.1/dist/cdn.min.js"
          defer
          integrity="sha384-KLv/Yaw8nAj6OXX6AvVFEt1FNRHrfBHziZ2JzPhgO9OilYrn6JLfCR4dZzaaQCA5"
          crossorigin="anonymous"></script>
```

Otherwise Alpine directives can be dropped into your standard Django or Jinja2 templates without any additional setup.

## Summary

The combination of Alpine and HTMX is very powerful and pushes the boundaries of what you can do with a "traditional" Django architecture. You can have much of the functionality and usability of the SPA model, without the associated fragility, complexity and cost. This is of particular interest to early stage startups, hobby developers and small SAAS companies who cannot afford a larger team of backend and frontend specialists. Futhermore these libraries have a very shallow learning curve, each consisting of only a couple dozen custom directives, that can be quickly mastered by a backend developer with a basic working knowledge of HTML and Javascript. You can even add HTMX and Alpine to an existing "legacy" Django project, sprinkling in a few actions here and there to improve the user experience.

Should you then throw away your React or Vue code and enjoy the simpler life? As always, the answer is "it depends". If you are building a very complex frontend with a lot of user interactions - a game for example, or something like Notion or Google Docs - a heavy frontend framework may be a better choice. The larger initial load doesn't matter too much as users will tend to keep the application tab open in their browser for a longer period. SEO might not be an issue either if users have to log into your app to access it. These frameworks also have healthy ecosystems, providing third-party libraries, tutorials and documentation, as well as a large hiring pool. It's also possible to use e.g. React for one small part of your site where it makes sense, while the rest of the site uses a multi-page architecture - a complex interactive dashboard, for example.

The problem however is not with the SPA architecture itself, but rather the current dominant mindset of SPA as the default paradigm for all web projects, rather than one possible approach among many others for careful consideration based on the requirements of the project and the skills of the development team. That's why libraries such as HTMX and Alpine are a great addition to your toolkit.
