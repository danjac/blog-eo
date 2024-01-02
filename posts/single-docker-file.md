---
title: The simplest Dockerfile for PAAS
description: Using a multistage Docker deployment for Dokku and other PAAS deployments
date: 2024-01-02
tags:
  - devops
  - django
  - docker
  - dokku
  - paas
layout: layouts/post.njk
---

For my side projects I usually have very simple needs. I typically don't need multi-server setups with a load balancer and a managed database, for example. Typically these little projects are for learning, or for personal use, with low traffic. Of course they may fall the Reddit or Hacker News Hug of Death, but so what?

My typical go-to has been [Hetzner](https://www.hetzner.com/). A shared VM with with 20-40 GB disk space and 2-4 GB RAM costs around 5 EUR a month, which is reasonable for a hobbyist project. Other options include [Fly.io](https://fly.io) or [Render](https://render.com); as of writing Heroku, once the go-to in the PAAS world, no longer provides free accounts, while the first two require a credit card upfront even for free offerings, which may or may work for you depending on your mileage.

For a Hetzner VM I typically use [Dokku](/posts/dokku). This is a great tool for a single-server deployment, as close as possible to the "push to deploy" simplicity of Heroku.

Dokku, as other PAAS offerings, can deploy by "guessing" the kind of deployment you have from your local files. For example if you have a **requirements.txt** file, Dokku will assume a Python application. If you have **package.json** it will try and run Node. This however gets a bit awkward when you have both Python and Node files, or some other combination, or if you have additional or specific steps you need to add to your build step.

However Dokku will also auto-detect a Dockerfile and use that to build the production image. The advantage of using Dockerfile is that you can add exactly the steps you need, and is also portable: if I want to move from Dokku to some other provider, whether PAAS or, say, a Kubernetes pod, you can re-use your Dockerfile. As each step or layer is cached it makes for a fast build.

So what do I need for my production build? My typical side project has the following requirements:

1. Tailwind
1. Javascript/Typescript
1. Django

The Django site uses [Whitenoise](https://whitenoise.readthedocs.io/en/latest/) to serve static files, but those static files need to be available in the image. So I need to build the static images inside the image itself, i.e. the steps have to be included in the Dockerfile.

The static files build steps in this case need to compile and compress the CSS and JS files. For Django we need to install dependencies. Once Django is installed, the static files have to be copied over to the right directory.

The simplest approach is to use a multi-staged Dockerfile. This allows you to base your image not one one, but multiple images.

What does this look like? The following example is adapted from a side project of mine:

```docker
  # Frontend

  FROM node:20-bookworm-slim AS frontend

  WORKDIR /app

  # Asset requirements

  COPY ./package.json /app/package.json

  COPY ./package-lock.json /app/package-lock.json

  RUN npm install

  # Build assets

  COPY . /app

  ENV NODE_ENV=production

  RUN npm run build-css && npm run build-js

  # Backend

  FROM python:3.12.1-bookworm AS backend

  ENV PYTHONUNBUFFERED=1

  ENV PYTHONDONTWRITEBYTECODE=1

  ENV PIP_DISABLE_PIP_VERSION_CHECK=1

  ENV PIP_ROOT_USER_ACTION=ignore

  WORKDIR /app

  COPY ./requirements.txt /app/requirements.txt

  RUN pip install -r /app/requirements.txt --no-cache-dir

  # Copy over files

  COPY . /app

  # Build and copy over assets

  COPY --from=assets /app/static /app/static

  # Collect static files for Whitenoise

  RUN python manage.py collectstatic --no-input --traceback
```

The first stage builds our frontend requirements. This uses a Node image (I prefer Bookworm generally for smaller images), installs requirements and builds the frontend dependencies. My **package.json** in this case has a couple directives to compile Tailwind (or PostCSS, SASS etc) into production-ready CSS, and another to build the Javascript using esbuild, vite or similar.

The second stage builds the backend: it's pretty straightforward, just install requirements for my Django application. But the final two steps are important to combine our frontend and backend into a single production deploy:

```docker
  COPY --from=assets /app/static /app/static

  RUN python manage.py collectstatic --no-input --traceback
```

The **COPY --from** directive pulls from our frontend stage. Note how each stage is named, so:

```docker
  FROM node:20-bookworm-slim AS frontend
```

So **COPY** will copy over from the **assets** directory in the **frontend** stage to our **backend** stage.

We can then run collectstatic, so Whitenoise can find these files:

```docker
  RUN python manage.py collectstatic --no-input --traceback
```

One problem here: usually running **manage\.py** in Django can cause problems when building an image, because you need for example the **SECRET_KEY** setting to run a Django process. I get around this using a default variable e.g. in my **settings\.py**:

```python
  SECRET_KEY = os.environ("SECRET_KEY", "django-insecure-some-random-key")
```

Some people prefer not to allow this at all, as it may lead inadvertently deploying an insecure key to production, and will do something like this:

```python
  try:
    SECRET_KEY = os.environ["SECRET_KEY"]
  except KeyError:
    raise ImproperlyConfigured("SECRET_KEY is missing!")
```

(Tools like [python-decouple](https://pypi.org/project/python-decouple/) and [django-environ](https://pypi.org/project/django-environ/) do this in a neater way).

You can however add a temporary secret key in your Dockerfile, as it will just be used when building the image, and not in production:

```python
  ENV SECRET_KEY="django-insecure-some-random-key"
  RUN python manage.py collectstatic --no-input --traceback
```

If however again you want to avoid setting any hard-coded secrets, even if temporary, you can use **ARG** to pass them as an option at build time. In Dokku you can do this with the [docker-options](https://dokku.com/docs/advanced-usage/docker-options/) commands:

```bash
  dokku docker-options add myapp build "--build-arg SECRET_KEY=my-secret"
```

And in your Dockerfile:

```docker
  ARG SECRET_KEY
  ENV SECRET_KEY=${SECRET_KEY}
```




