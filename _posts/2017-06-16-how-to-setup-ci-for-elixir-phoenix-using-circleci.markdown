---
layout: post
title:  "How to setup CI for Elixir/Phoenix using circleci"
date:   2017-06-16 22:49:00 +0530
published: true
categories:
  - circleci
  - ci
  - elixir
  - phoenix
---

Having automated tests and CI is very good for the overall health of a project.

circleci provides a usable few plan, which is awesome! I set up my phoenix project using it.
It is actually pretty straight forward to setup circle ci. Just add the following file to your repository:


### ./circleci/config.yml


```yaml

version: 2
jobs:
  build:
    environment:
      - MIX_ENV: test
      - DW_DROPBOX_APP_KEY: dummy
      - DW_DROPBOX_APP_SECRET: secret
    working_directory: ~/dw
    docker:
      - image: elixir:1.4.4
      - image: postgres:9.4.1
        environment:
          POSTGRES_USER: ubuntu
    steps:
      - checkout
      - run: mix local.hex --force
      - run: mix local.rebar --force
      - run: mix deps.get
      - run: mix ecto.create
      - run: mix test

```

Adding this file is all you need to have circle ci build your projects on every git push to bitbucket/github.
