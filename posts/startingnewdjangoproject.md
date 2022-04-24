---
title: Starting a New Django Project
description: Starting a new Django project
date: 2022-04-24
tags:
  - django
  - python
  - web development
layout: layouts/post.njk
---

![Green Field](/img/greenfield.jpg)

## Introduction

Occasionally I get to start over with a new Django project. Usually this is just some side project: *very* occasionally I get to build a greenfield project for someone else (protip for new developers: 99.99% of your career you'll be inheriting someone else's project, for better or worse).

If it's just a quick throwaway, for a tutorial or proof-of-concept, I just type `django-admin.py startproject` and go with all the defaults. I won't need anything more than Sqlite and I definitely won't need Docker. Maybe a virtualenv and that's that.

But say it's a serious project that is likely to have legs. What does a "good" project skeleton look like?

## Cookiecutters

At this point some people will recommend using a cookiecutter, and the best supported and maintained right now is [Daniel Roy Greenfeld's Django cookiecutter](https://github.com/cookiecutter/cookiecutter-django/). If you have never built a large Django project before you could do far worse than use this. It comes with some good defaults. Personally I find it a little too large, and it has a lot of artefacts I don't need, but I still use it as a reference for current thinking about best practices in Django.

I also do not maintain my own cookiecutter. I've tried a couple times, but they're a pain to maintain. You want to keep adding new things to the cookiecutter to reflect your learning in your Django projects, and you also want to keep up with latest changes in the ecosystem - for example, a best-of-breed library falls out of favour in place of the next hot thing. Over time the cookiecutter drifts away from your latest projects and thinking.

I would probably keep a cookiecutter if a) there were other maintainers who could help keep it up to date or b) I was running an agency where I make new projects every other week and a cookiecutter saves perhaps days of work each time, and there is a need to maintain a base level of quality and consistency. At present though I just start with a plain Django project and make the changes I need manually.

## Project configuration

Typically you'll find:

* .dockerignore
* .editorconfig
* .flake8 (because Flake doesn't support pyproject.toml)
* .gitignore
* .npmrc
* .pre-commit-config.yaml
* .prettierrc
* package.json
* pyproject.toml
* requirements.in
* requirements.txt

My pre-commit file typically includes (in addition to standard whitespace checking etc):

* Bandit
* Black
* Flake8
* absolufy-imports
* djhtml
* isort
* mypy
* prettier

I have tried mypy with `django-stubs`, but found it a massive pain to work with due to need to run it inside the Docker container (among other problems), so I just use mypy with these settings:

```toml
  [mypy]
  python_version = 3.10
  check_untyped_defs = false
  ignore_missing_imports = true
  show_error_codes = true
  warn_unused_ignores = false
  warn_redundant_casts = false
  warn_unused_configs = false
  warn_unreachable = true
```

Perhaps not as comprehensive as `django-stubs` but good enough to provide some benefit to typing.

## Settings

My typical settings structure will look something like this:

```
  myproject
    - settings
        base.py
        local.py
        production.py
        test.py
    urls.py
    wsgi.py
```

Some people like to have an extra level or use a `config` package or something like that. Personally I dislike the extra typing that involves.

To keep settings maintainable I use [django-environ](https://pypi.org/project/django-environ/) to use environment variables as much as possible, and [django-split-settings](https://pypi.org/project/django-split-settings/) to keep inter-settings imports nice and tidy. For example in `local.py` instead of this:

```python
  from .base import *

  INSTALLED_APPS = INSTALLED_APPS + ["debug_toolbar"]
```

we have:

```python

  from split_settings.tools import include

  from myproject.settings.base import INSTALLED_APPS

  include("base.py")

  INSTALLED_APPS = INSTALLED_APPS + ["debug_toolbar"]
```

## Templates

Generally I avoid per-app templates, but keep the templates all in one place under the top-level directory. Keeping them in one easily-accessible place is nice and consistent, particularly if non-Django developers (say frontend people) want to work on them and don't particularly want to have to hunt around in different apps trying to find the right file (the same goes, of course, for static files).

I've gone back-and-forth on naming individual templates and subdirectories, particularly regarding partials. For a while I used the underscore convention for example `_article.html` as opposed to a non-partial `article.html`. Nowadays I prefer to move partials under an `includes` subdirectory and avoid the underscore naming. This is just a personal preference thing, but it avoids a directory becoming too large with similarly named files. The top level templates directory will have a "junk drawer" `includes` directory (in addition to specific includes for things like pagination templates) and individual apps will have their own `includes`:


```
  myproject
    + myproject
    - templates
        base.html
        - django
            - forms
                default.html
        - includes
            sidebar.html
        - pagination
            pagination_links.html
            ...
        - articles
            article.html
            - includes
              - article.html
```

Rule of thumb: if I have to `{% raw %}{% include %}{% endraw %}` a template (or access it using an inclusion template tag) it goes in the relevant `includes` subdirectory, unless the include has a specific function like pagination, forms etc.

## Static files

Static files also live under the top directory:

```
  myproject
    - myproject
    - static
        + css
        + dist
        + img
        + js
```

The `dist` directory contains any files generated by whatever frontend build system (django-compress, esbuild, webpack etc) such as minified/processed CSS and Javascript/sourcemap files. These days I tend to start with [a more lightweight frontend](/posts/anatomyofdjangohtmxproject/) but if I'm building a full SPA the static files will probably live in their own frontend directory at the top level of the project (or an entirely different repo).

I prefer Tailwind over Bootstrap, Bulma and other CSS frameworks, at least as a starter default, so I'll probably have a `tailwind.config.js` in the top directory as well.

## Local apps

Django apps are a bone of contention for even experienced developers. First of all the word "app" screams "I was a framework designed pre-2007" as the very word has changed in meaning not only in the tech world but in mainstream culture. Perhaps if Django were invented today we'd use something else; the Elixir framework Phoenix has "contexts" for example, but maybe "domains" would be more accurate (although we would then go into the weeds of Domain-Driven Development). Nevertheless, the hardest part about apps is not so much what we call them but deciding on their granularity. Some developers like to make Django more like Rails or Laravel and have a giant single app with separate packages for models, views and so on. I personally like the concept of apps though and prefer to keep them relatively small, with a few models per app.

In any case your apps will change during the lifetime of the project. However I know I'm probably going to need users and I'll probably need somewhere to put any code that's not going to fit anywhere else (or is not particularly business domain-specific): a "junk drawer". You can call your junk drawer app whatever you want, I like "common".

```
  myproject
    - myproject
        + common
        + settings
        + static
        + users
        urls.py
        wsgi.py
```

Some projects I've seen have an `apps` directory but personally I find this redundant, especially if using absolute imports. I also have a personal aversion to calling packages and modules `utils`: if I have a couple of functions that do networking stuff, for example, I'll make a `networking` module rather than keep them in a `utils` module.

## Preferred libraries

I've already mentioned `django-environ` and `django-split-settings`. Other favourites include:

* [dj-database-url](https://pypi.org/project/dj-database-url/)
* [django-allauth](https://pypi.org/project/django-allauth/)
* [django-cachalot](https://pypi.org/project/django-cachalot/)
* [django-extensions](https://pypi.org/project/django-extensions/)
* [django-model-utils](https://pypi.org/project/django-model-utils/)
* [django-redis](https://pypi.org/project/django-redis/)
* [django-widget-tweaks](https://pypi.org/project/django-widget-tweaks/)
* [psycopg2-binary](https://pypi.org/project/psycopg2-binary/)
* [redis](https://pypi.org/project/redis/)
* [whitenoise](https://pypi.org/project/whitenoise/)

For production I'll probably throw in:

* [django-permissions-policy](https://pypi.org/project/django-permissions-policy/)
* [django-anymail](https://pypi.org/project/django-anymail/)
* [gunicorn](https://pypi.org/project/gunicorn/)
* [sentry-sdk](https://pypi.org/project/sentry-sdk/)

And for local development:

* [django-debug-toolbar](https://pypi.org/project/django-debug-toolbar/)

I'm a big fan of pytest (I'm aware some developers are less so). But as I am, I also include these libraries:

* [pytest](https://pypi.org/project/pytest/)
* [coverage](https://pypi.org/project/coverage/)
* [pytest-django](https://pypi.org/project/pytest-django/)
* [factory-boy](https://pypi.org/project/factory-boy/)
* [pytest-cov](https://pypi.org/project/pytest-cov/)
* [pytest-forked](https://pypi.org/project/pytest-forked/)
* [pytest-mock](https://pypi.org/project/pytest-mock/)
* [pytest-randomly](https://pypi.org/project/pytest-randomly/)
* [pytest-xdist](https://pypi.org/project/pytest-xdist/)

If I'm using [HTMX](https://htmx.org) (and for new projects that's increasingly the case) I'll also add [django-htmx](https://pypi.org/project/django-htmx/).

For queuing, as mentioned, I go with `rq` over Celery unless I have bigger requirements, and starter projects tend to have pretty small requirements.

Regarding requirements: some people recommend splitting these up into local, production, test and so on, but I've found that more micromanagement that I like and it's easy to add a requirement in the wrong place and end up with a broken build. It's not the worst thing if your production has to install pytest libraries, for example, but you can easily miss or delete an important library from your production requirements and have everything work in your CI/CD pipeline only to have a nasty surprise at the end.

In addition I tend to use Heroku or [Dokku](https://github.com/dokku/) for early-stage projects, and these work out of the box with a plain `requirements.txt` file.

That's not so say a more complex requirements setup is better for a larger and more complex project, but for a *starter* project (the subject of this article) I want to keep it simple as possible.

I've tried [Poetry](https://github.com/python-poetry) a few times, but in general I've found it [slow](https://github.com/python-poetry/poetry/issues/2094) and error-prone, particularly inside Docker environments. I see the appeal, and I hope to make it my go-to some day, but right now I find it more trouble than it's worth. Instead I use [pip-tools](https://github.com/jazzband/pip-tools) to generate and update my `requirements.txt` file from a `requirements.in` file.

## Docker

One other library I enjoy for local development - particularly as I tend to start with Heroku/Dokku for early-stage deployments - is [Honcho](https://honcho.readthedocs.io/en/latest/). It makes it easier to add a development Procfile and wrap all my local environment into a single Docker image:

```yaml
  services:
      honcho:
          build:
              context: .
          environment:
              DATABASE_URL: postgres://postgres:password@postgres:5432/postgres
              EMAIL_HOST: mailhog
              EMAIL_PORT: 1025
              REDIS_URL: redis://redis:6379/0
              SECRET_KEY: seekrit!
          restart: on-failure
          privileged: true
          tty: true
          stop_grace_period: '3s'
          logging:
              options:
                  max-size: '100k'
                  max-file: '3'
          ports:
              - '8000:8000'
          depends_on:
              postgres:
                  condition: service_healthy
              redis:
                  condition: service_healthy
          volumes:
              - ./:/app
              - /app/node_modules
          command: [
              'honcho',
              'start',
              '-f',
              './Procfile.local'
          ]
```

The `Procfile.local` file looks something like this:

```
  web: python manage.py runserver 0.0.0.0:8000
  worker: python manage.py rqworker mail default
  watcher: npm run watch
```

This also means I can keep my local image in a small, simple `Dockerfile`. This one includes both Python and frontend (Node/npm):

```docker
  FROM python:3.10.4-buster

  ENV PYTHONUNBUFFERED 1
  ENV PYTHONDONTWRITEBYTECODE 1
  ENV PYTHONFAULTHANDLER=1
  ENV PYTHONHASHSEED=random

  ENV NODE_VERSION 17.9.0

  RUN curl "https://nodejs.org/dist/v$NODE_VERSION/node-v$NODE_VERSION-linux-x64.tar.xz" -O \
      && tar -xf "node-v$NODE_VERSION-linux-x64.tar.xz" \
      && ln -s "/node-v$NODE_VERSION-linux-x64/bin/node" /usr/local/bin/node \
      && ln -s "/node-v$NODE_VERSION-linux-x64/bin/npm" /usr/local/bin/npm \
      && ln -s "/node-v$NODE_VERSION-linux-x64/bin/npx" /usr/local/bin/npx \
      && rm -f "/node-v$NODE_VERSION-linux-x64.tar.xz"

  WORKDIR /app

  COPY ./requirements.txt ./requirements.txt

  RUN pip install -r ./requirements.txt

  COPY ./package.json ./package.json
  COPY ./package-lock.json ./package-lock.json

  RUN npm cache clean --force && npm ci
```

A more complex project might require using multi-stage builds, or multiple Docker images/containers, but again KISS is the principle until I know I need that complexity.

Note that this is a Docker set up for **local development**. Nowadays there are so many approaches to production deployments that it's very difficult to come up with generalized advice, especially around containerized deployments. You could go with anything from a simple PAAS deployment like Heroku, Dokku or [Render](https://render.com) up through various AWS or Google Cloud environments or a more complex multi-cloud Kubernetes-based set up (or even your own on-prem hardware), and a lot depends on requirements, experience and budget.

## Scripts

I typically have a top-level `bin` directory with simple Bash scripts to access some common things inside Docker containers:

```
  - bin
      manage
      npm
      pytest
```

So for example `bin/manage` will look like:

```bash
  #!/usr/bin/env bash

  set -euo pipefail

  docker-compose exec honcho ./manage.py $@
```

This saves on a lot of tedious typing, so for example I can do `./bin/manage shell` instead of `docker-compose exec honcho ./manage.py shell`.

## Makefile

I'll usually make a simple `Makefile` at some point with common things e.g.

* `make build`: to build containers
* `make test`: run unit tests
* `make up` or `make start`
* `make down` or `make stop`
* `make shell`: start Django shell

Again, I like a smooth local development environment with minimal typing, especially for things I have to do over and over.

Source code for this article [here](https://github.com/danjac/myproject).
