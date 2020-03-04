---
title: "Docker Compose + Ruby on Rails 6"
date: 2020-03-02T13:20:07-05:00
draft: false
tags: [
  "tutorial",
  "rails",
  "ruby",
  "docker"
]
---
Recently, I was trying to set up a Rails 6 app with Docker Compose.

Conveniently, there are [docs](https://docs.docker.com/compose/rails/) provided
by Docker to set up a Rails app with Postgres.

However, this guide targets Rails 5 -- I wanted Rails 6! Luckily, the guide
required little adjustment and was easy to adapt.

I'll keep this tutorial short and concise -- I'd like to reproduce more of the
original guide here but will refrain since there's no free license on it.
I'll use the same headings as the
[original tutorial](https://docs.docker.com/compose/rails/) -- just follow
along and I'll tell you when you need to add something.


## Define the project
Inside the Dockerfile, we need to add to the `RUN` command that reads:

```
RUN apt-get update && apt-get install -y nodejs postgresql-client # bad! we need yarn!
```

We need to install `yarn` as well so the `webpacker` gem introduced in Rails 6
can be installed.

We do this by adding the yarn repository to our sources and installing yarn:

```
RUN curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | apt-key add -
RUN echo "deb https://dl.yarnpkg.com/debian/ stable main" | tee /etc/apt/sources.list.d/yarn.list
RUN apt-get update && apt-get install -y nodejs postgresql-client yarn
```

We also need to change the bootstrap `Gemfile` from the one provided so we use Rails 6:
```
source 'https://rubygems.org'
gem 'rails', '~>6.0.2.1'
```

I picked the most recent version at the time of installation -- Rails 6.0.2.1.

Follow the original guide's directions for creating a `Gemfile.lock` and
providing `entrypoint.sh`.

The `docker-compose.yml` file should be very similar -- however, when using the
most recent version of the `postgres` image, we'll need to add a default password.

Add an `environment` key to the `db` service to look like this:

```
services:
  db:
    ...
    environment:
      - POSTGRES_PASSWORD=<password>
    ...
```

Should you use a separate, untracked `.env` file or some other secret
management solution? Absolutely! But for my development purposes, not worth
the effort.

## Build the project

You shouldn't have to change anything here! Pat yourself on the back.
Get a glass of water -- dehydration is bad for the creative process.

## Connect the database

You should only have to provide the password you set up earlier to `config/database.yml`.

```
default: &default
  ...
  password: <password>
```

Follow the rest of the instructions with no modifications.

## View the Rails welcome page!

You should now have a Rails 6 app working! Now go forth and Rails!

