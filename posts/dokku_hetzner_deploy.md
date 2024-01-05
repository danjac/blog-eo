---
title: Deploy a Django application with Hetzner and Dokku
description: Steps for building a micro-Heroku in 15 minutes.
date: 2024-01-05
tags:
  - dokku
  - web development
  - devops
  - django
  - hetzner
layout: layouts/post.njk
---

In [previous](/posts/dokku) [posts](/posts/starting-docker-file) I talked about [Dokku](https://dokku.com), a self-hosted Platform as a Service (PAAS) solution that you can run on Hetzner.

This is a great solution for a single developer, when you want to launch a hobby side project or small business and you don't have a big budget. Here I go through the steps to launching your own micro-PAAS in just 15-20 minutes!

## Prep

For our example, we are going to deploy a Django application, `photogram`. We are going to host our site on the domain `photogram.com`.

To make this project "Dokku-aware" you should have at least an `app.json` and a `Procfile`. For a bare minimum example our `app.json` file looks like this:

```json
  {
      "name": "My little SAAS",
      "formation": {
          "web": {
              "quantity": 1
          }
      },
  }
```

and the `Procfile`:

```
  release: ./release.sh
  web: gunicorn -c ./gunicorn.conf.py
```

If you have used Heroku, the `Procfile` should be familiar.

The release process, which is triggered on each deployment, points to a `release.sh` file that runs migrations, does some system checks etc:

```bash
  #!/usr/bin/env bash

  set -o errexit

  ./manage.py check --deploy

  ./manage.py migrate

  ./manage.py clear_cache
```

I like to point `gunicorn` at a Python module:


```python
  import multiprocessing

  # https://docs.gunicorn.org/en/stable/configure.html#configuration-file

  wsgi_app = "project.wsgi"

  accesslog = "-"

  workers = multiprocessing.cpu_count() * 2 + 1
```

The neat thing about using a Python module instead of a config file or command line options is that you can calculate some things dynamically, e.g. the number of workers based on available CPU.

You can add more processes to your `app.json` and/or `Procfile` as needed, for example if you want to run celery, or schedule cron jobs. See the [Dokku](https://dokku.com) documentation for details.

I prefer to handle Dokku deployments using Docker rather than buildpacks. For details on building an optimal `Dockerfile` for Dokku deployments, see [my previous article](/posts/dokku).

## Create a Hetzner project

### Build your project

### Add firewalls

### Set up DNS

## Install Dokku

### Installing Dokku

### Add plugins

### Create your Dokku app

### Set up PostgreSQL

### Set up Redis

### Set up LetsEncrypt

## Deploy your code

### Push your code to Dokku

### Set environment variables

## Finishing up

### Create a superuser
