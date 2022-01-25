---
title: Django, HTMX and Alpine
description: The old is new again - a better way to build web apps?
date: 2022-01-24
tags:
  - web development
  - python
  - django
layout: layouts/post.njk
---

![The Winter Panorama, Champéry, Switzerland](/img/joel-jasmin-forestbird-Zi9yS95gDUQ-unsplash.jpg)

Photo by [Joel & Jasmin Førestbird](https://unsplash.com/@theforestbirds?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/alps?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)
Over the past year or so I've been closely following a trend in web development that eschews the complexity of the Single Page Application (SPA) architecture, breathing new life into an older pattern in order to build modern web applications quicker, cheaper and simpler.

This pattern has been referred to "HTML Over The Wire". To give an idea of the principles involved, let's imagine we want to build a web app, perhaps a data-driven SAAS targeting small businesses.

In a Single Page Application (SPA) architecture, the frontend would be built using a Javascript framework such as React or Vue, which would talk back and forth to a backend API using JSON payloads. This is simplifying things a lot, but that's the basic idea.

So our SAAS app, using an SPA model, would consist of a backend API - we're using Django, so we also add the Django Rest Framework (DRF) to take care of routing and serialization. Our API receives and sends JSON payloads. Our frontend is built in React, again with additional libraries as needed for things like components, form validation and state management.

This model has some advantages. It allows separation of concerns: if we wanted to serve not just web pages but, say, an Android app, the Android app could talk to the same API as our web application. The backend logic can be changed completely - in theory you could decide one day to migrate from Django to Rails, say, and as long as your API stays the same the frontend doesn't need to know or care. This enables developers in different teams and disciplines - backend and frontend specialists - to work independently. Furthermore, a frontend framework like React can - in theory - provide a smoother experience to the end-user, without full page loads and slow and janky form processing experiences. It's easy to see why the SPA model has become the de-facto pattern of web development in the past decade.


