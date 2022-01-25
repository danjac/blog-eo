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
Over the past year or so I've been closely following a trend in web development that eschews the complexity of the Single Page Application (SPA) architecture, breathing new life into an older pattern in order to build modern web applications quicker, cheaper and simpler.

This pattern has been referred to "HTML Over The Wire". To give an idea of the principles involved, let's imagine we want to build a web app, perhaps a data-driven SAAS targeting small businesses.

In a Single Page Application (SPA) architecture, the frontend would be built using a Javascript framework such as React or Vue, which would talk back and forth to a backend API using JSON payloads. This is simplifying things a lot, but that's the basic idea.

So our SAAS app, using an SPA model, would consist of a backend API - we're using Django, so we also add the Django Rest Framework (DRF) to take care of routing and serialization. Our API receives and sends JSON payloads. Our frontend is built in React, again with additional libraries as needed for things like components, form validation and state management.

This model has some advantages. It allows separation of concerns: if we wanted to serve not just web pages but, say, an Android app, the Android app could talk to the same API as our web application. The backend logic can be changed completely - in theory you could decide one day to migrate from Django to Rails, say, and as long as your API stays the same the frontend doesn't need to know or care. This enables developers in different teams and disciplines - backend and frontend specialists - to work independently. Furthermore, a frontend framework like React can - in theory - provide a smoother experience to the end-user, without full page loads and slow and janky form processing experiences. It's easy to see why the SPA model has become the de-facto pattern of web development in the past decade.

The SPA model however also comes with a lot of downsides. Logic such as form validation has to be duplicated between client and server. You have to manage two applications, perhaps running in separate domains, with solutions such as [CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS). A bug in your Javascript code may render not merely a semi-functional site, but a blank page with no clue for the developer or user on how to fix. SEO is more problematic as Javascript-rendered sites are opaque to search engine crawlers and social media sharing URLs. A web page might initially load more quickly, but the user is left looking at numerous spinning gifs while half a dozen API calls load parts of the page. There are solutions to these issues, from SSR to GraphQL, but they entail further complexity and more dependencies.

All of this additional complexity translates to a higher cost in time and money to get your product to market, and a higher cost in salary or consultant fees as you have to hire larger teams of specialists. The complexity also creates a larger surface area for bugs and potential security issues (the sheer number of NPM modules required to run an average SPA frontend alone means that sooner or later, [you are going to inadvertently import insecure or even malicious code](https://arstechnica.com/information-technology/2021/09/npm-package-with-3-million-weekly-downloads-had-a-severe-vulnerability/), no matter how much auditing you do). Moreover these dependencies amount to huge initial loads - perhaps irrelevant to developers working with expensive, high-end laptops and excellent high-speed internet, but not so much fun to someone using an older mobile device in a rural area with poor connectivity.

However, we have to acknowledge that SPAs, no matter their costs and disadvantages, became popular because they solved problems with the old server-side rendering model, for example full page reloads on every button or link click or the tiresome post-validate-repost cycle of form validation. Solutions for these did exist in the pre-SPA era. One common pattern used [jQuery.load()](https://api.jquery.com/load/) to fetch chunks of HTML and inject them into the DOM. This was fine for isolated cases, but became difficult to manage in larger and more complex apps. More fully-fledged frameworks (or their [ecosystems](ecosystems)) such as Angular and React provided more of, well, a framework for handling complex DOM manipulations.


This pattern however didn't die out completely during the Javascript framework boom of the 2010s. A libraries were released which provided easier and more structured means of handling AJAX and DOM interaction, including:

* [PJAX](https://github.com/defunkt/jquery-pjax)
* [Turbolinks](https://github.com/turbolinks/turbolinks)
* [Intercooler](https://github.com/bigskysoftware/intercooler-js)
* [Unpoly](https://unpoly.com/)

I should also mention two other more framework-specific projects: [Laravel Livewire](https://laravel-livewire.com/) and [Phoenix Channels](https://www.phoenixframework.org/).

Turbolinks was subsumed into the [Hotwire](https://hotwire.dev) project, developed by the Rails team. A successor to Intercooler, also developed by Carson Gross, was released in 2021: [HTMX](https://htmx.org).

## HTMX

HTMX makes it easy to progressively enhance a traditional server-side rendered web application through a set of custom HTML attributes. For example, suppose we have a "Subscribe" button on a web site, for example if you wanted to subscribe to a Youtube channel. When this button is clicked,  the user is subscribed to the channel and the button text should be updated to "Unsubscribe".

A traditional server-rendered application might look like this:

``` html
  <form method="POST" action="/subscribe/12345/">
    <button type="submit">Subscribe</button>
  </form>
```

Clicking the button submits our form to the server in a POST action. The server code would likely have to respond with an HTTP redirect back to the same page, forcing a full page reload. A button in of itself can't do a POST, so we have to wrap it in a `<form>` tag.

A full page reload as well as being a janky user experience is inherently wasteful of server resources: for example, suppose you need to render a navbar, footer, page details and so forth, requiring additional database calls? Fine for an initial page load, but doing all that just to update a button text is a lot of wasted resources.

HTMX allows us to handle this in a more graceful fashion. Using custom attributes with an *hx-** prefix (HTML purists can also use *data-hx-**) the button itself can handle the POST:

```html
  <button hx-post="/subscribe/12345/"
          hx-target="this"
          hx-swap="outerHTML">Subscribe</button>
```

The `hx-post` attribute means "send an HTTP POST to this URL". `hx-target` provides the DOM to be swapped (it can be any valid DOM selector, such as an ID or class; `this` is a special designator meaning "this element"), and `hx-swap` the actual DOM manipulation to be done on the result - in this case, replace the entire `<button>` with whatever HTML is returned. These three attributes alone can do a lot of lifting; HTMX has a [couple dozen such directives](https://htmx.org/reference/) in its toolbox, providing all sorts of AJAX-related functionality without writing a single line of Javascript.

The URL - it doesn't matter what language it is, HTMX will work with any server language you like, from Python or PHP to Go or Rust - should then return an HTML snippet, something like this:

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

In addition to small AJAX interactions, you can use HTMX to handle full page navigation, using the `hx-boost` feature. This works in a similar way to Turbolinks/Hotwire: you return full pages from the server, but the differing DOM elements in the `<body>` are swapped rather than doing a full reload. This provides a smoother experience for the user when navigating around the site, similar to the experience of using an SPA. The easiest way to get started is to build a traditional web application in whatever backend framework or language you want, and gradually add HTMX attributes to links, forms and other elements as needed.

HTMX also provides support for "push" techniques such as [Server-Sent Events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events) (SSE) and WebSockets. Hotwire and

## HTMX and Django

Integration with Django is quite straightforward and does not require anything more than including the CDN in your base template. One thing to bear in mind is that HTMX provides a number of [request headers](https://htmx.org/reference/#request_headers) so you can provide more efficient and targeted responses.

For example in a view you might wish to return an HTML snippet if you know the request originated from an HTMX action, or a full page response in other cases:

```python
  def my_view(request):
      if request.headers.get("HX-Request"):
          return TemplateResponse(request, "_some_snippet.html")
      else:
          return TemplateResponse(request, "full_page.html")
```

You might also wish to add HTMX-specific headers to your outbound response. For example you can return an `HX-Redirect` header to instruct HTMX to do a client-side redirect to another location.

It's recommended however to use the [Django-HTMX package](https://github.com/adamchainz/django-htmx) developed by Adam Johnson. This provides a Django middleware and other useful helpers to make Django and HTMX integration more seamless. For example, if you install the django-htmx middleware the above code can be rewritten:


```python
  def my_view(request):
      if request.htmx:
          return TemplateResponse(request, "_some_snippet.html")
      else:
          return TemplateResponse(request, "full_page.html")
```

## Alpine

Of course, Javascript can do a lot more than handle AJAX interactions. For example, you might have modals, drop-down menus, "flash" messages and other interactions and effects that are a bit more tricky to do than using CSS. In the past you might have used vanilla JS or jQuery; if you are using a framework like React you'll likely be using this framework for doing this kind of work.

Thankfully a number of libraries have arisen to do this kind of in-page work, providing an easier way to organize your frontend code without requiring a large frontend framework. One such solution, again part of the Hotwire family, is [Stimulus](https://stimulus.hotwired.dev/). Another is [Alpine.js](https://alpinejs.dev/). Another framework developed by Carson Gross and others is [Hyperscript](https://hyperscript.org/).

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

Thus when the `@click` event is triggered, the class `text-red` is applied.

Again as with HTMX, you can install Alpine.js using a CDN:

```html
  <script src="https://unpkg.com/alpinejs@3.7.1/dist/cdn.min.js"
          defer
          integrity="sha384-KLv/Yaw8nAj6OXX6AvVFEt1FNRHrfBHziZ2JzPhgO9OilYrn6JLfCR4dZzaaQCA5"
          crossorigin="anonymous"></script>
```

Otherwise Alpine directives can be dropped into your standard Django or Jinja2 templates without any additional setup.

## Summary

The combination of Alpine and HTMX is very powerful and pushes the boundaries of what you can do with a "traditional" Django architecture. You can have much of the functionality and usability of the SPA model, without the associated fragility, complexity and cost. This is of particular interest to early stage startups, hobby developers and small SAAS companies who cannot afford a larger team of backend and frontend specialists.

Should you then throw away your React or Vue code and enjoy a simpler life? As always, it depends. While there is a great deal that can be done with HTMX and Alpine, a JS framework might be a good choice for very complex applications, or where you need very fast response times: think of Google Docs for example or an online game.

There is a very good argument in the article [The Single-Page-App Morality Play](https://www.baldurbjarnason.com/2021/single-page-app-morality-play/) which I strongly encourage you to read as it has a great insight not only into the pitfalls and dangers of the SPA paradigm but also the realities of modern web development. An SPA in the hands of a well-functioning organization and team with enlightened management and highly-skilled and talented developers will indeed be a joy to use and will probably represent the pinnacle of what can be done in with modern browser technology. But the fact is that most of us are not talented but, charitably, mediocre-to-decent and we working not for enlightened management but mediocre-to-decent managers (who then report to their mediocre-to-decent managers). Moreover, all of us, no matter how individually talented we may be, have to deal with difficult customers, tight deadlines, reduced budgets, employee churn and many other factors that act against our efforts at code and UX quality, and things get launched before they can be exhaustively tested and reviewed.

Not only do solutions like Alpine and HTMX raise the bar on what is doable with the "traditional" multi-page application paradigm, but they provide guard-rails and restraints on wildly ambitious attempts at poorly scoped, overly complex applications, forcing developers to work *with* the browser than against it. A functional, stable and accessible web page should be like a pyramid, with a solid base of HTML at the bottom, followed by a layer of CSS and Javascript at the top. SPAs instead try to invert this pyramid - and an upside-down pyramid is hardly the most stable of architectures.
