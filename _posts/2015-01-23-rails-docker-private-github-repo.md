---
layout:     post
title:      Rails Dockerized app using a private github repo
date:       2015-01-23 13:09:00
categories: docker
---

I've been playing with docker for a while, and I've been trying to use
to it with a rails app in production. One of the problems that I've
found was running bundler with a gem pointing to a private repository.

### 1. Create a ssh key to use with your docker containers

### 2. Set your public key on github

### 3. Base Dockerfile

Here is the base Dockerfile I use

```
FROM ruby:2.1.1
MAINTAINER JP Colomer <jpcolomer@gmail.com>
RUN apt-get update && apt-get install -y build-essential

#Copy ssh key to clone private github repo
RUN mkdir -p /root/.ssh
ADD docker_id /root/.ssh/id_rsa
RUN chmod 700 /root/.ssh/id_rsa
RUN echo "Host github.com\n\tStrictHostKeyChecking no\n" >> /root/.ssh/config
```

### 4. App Dockerfile

```
FROM base
MAINTAINER JP Colomer <jpcolomer@gmail.com>

RUN mkdir /app
WORKDIR /app

ADD Gemfile /app/Gemfile
ADD Gemfile.lock /app/Gemfile.lock

RUN bundle install --without test
```

Finally, when I want to run the docker container I add my app folder as
a volume on /app container's folder. I do this for
developing purposes.

In production I would probably create a third
Dockerfile and use the previous image as base. Something like this:

```
FROM app
MAINTAINER JP COLOMER <jpcolomer@gmail.com>

ADD . /app
```
