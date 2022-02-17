---
title: Using custom Django URL path converters
description: How to use a custom converter in your URLs
date: 2022-02-17
tags:
  - django
  - python
  - web development
layout: layouts/post.njk
---

![Honeycomb](/img/luemen-rutkowski-Buzk4BGDKGg-unsplash.jpg)

Photo by <a href="https://unsplash.com/@lulusphotography?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Luemen Rutkowski</a> on <a href="https://unsplash.com/s/photos/labyrinth?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>

You will very likely have seen Django's standard converters when building your URL patterns.

For example:

```python
  # urls.py

  from django.urls import path

  from myapp.views import post_detail

  urlpatterns = [
      path('posts/<int:post_id>/', post_detail, name="post_detail"),
  ]
```

The parameter *post_id* will be automatically converted to an `int` in your view args:


```python
  def post_detail(request, post_id: int):
      ...
```

This functionality is handled by a converter, in this case the `IntConverter`, mapped to the namespace *int*:


```python
  class IntConverter:
      regex = '[0-9]+'

      def to_python(self, value):
          return int(value)

      def to_url(self, value):
          return str(value)
```

The converter class must provide a `to_python()` method that takes a string and converts it to some other value (in this case an `int`) and a `to_url()`  method that takes a Python value and converts it into a URL path-friendly string.  You also need a `regex` property that allows Django's router to identify this converter when parsing the inbound URL.

You can see other examples [here](https://docs.djangoproject.com/en/2.2/_modules/django/urls/converters/).

Suppose you want to make your own custom converter? Let's say you have a use case where you want to add a date string to your path, and have Django convert that to a `datetime.date` instance as a view parameter. For example:

```bash
  http://example.com/posts/archive/2022-02-17/
```

The string *2022-02-17* is automatically converted to a `datetime.date` object in your view:

```python
  def post_archive(request, archive_date: date):
      ...
```

To get this to work, we need to create a custom converter class, and then register it with Django:

```python
  # urls.py

  from django.urls import path
  from datetime import date, datetime

  from myapp.views import post_detail, post_archive

  class DateConverter:
      regex = r"\d{4}-\d{1,2}-\d{1,2}"
      format = "%Y-%m-%d"

      def to_python(self, value: str) -> date:
          return datetime.strptime(value, self.format).date()

      def to_url(self, value: date) -> str:
          return value.strftime(self.format)

  register_converter(DateConverter, "date")

  urlpatterns = [
      path('posts/<int:post_id>/', post_detail, name="post_detail"),
      path('posts/archive/<date:archive_date>/', post_archive, name="post_archive"),
  ]
```

The `DateConverter` class is registered with the `register_converter()` function with the namespace "date". The regex will look for the pattern *YYYY-MM-DD* in the URL, and the parameter will be automatically converted into a `datetime.date` object.

The `to_url()` method on the other hand works in reverse. For example you can write:

```python
  from datetime import date
  from django.urls import reverse

  reverse("post_archive", args=[date(year=2022, month=2, day=17)])
```

and you will get the output */posts/archive/2022-02-17/*.

One further note: if the `to_python()` method raises a `ValueError`, Django will raise a `404 NOT FOUND`.  For example, if you try and create a `datetime` from an invalid date string, `strptime()` raises a `ValueError`:

```python
  from datetime import datetime
  datetime.strptime("2022-2-33", "%Y-%m-%d")
```

So this URL, even though it matches the `DateConverter` regex, will still raise a 404:

```bash
  http://example.com/posts/archive/2022-02-33/
```

Therefore take care that your `to_python()` method either implicitly or explicitly raises a `ValueError` if the conversion fails.
