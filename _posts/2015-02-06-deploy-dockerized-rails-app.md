---
layout:     post
title:      Deploy a dockerized Rails app with SSL
date:       2015-02-06 13:09:00
categories: docker
---

I've been using Docker in production lately. 
For a rails app I have a docker container and then I link it to a nginx
container.

Here is the Dockerfile I use for my rails app.

```
FROM phusion/passenger-ruby21:0.9.10
MAINTAINER JP Colomer <jpcolomer@gmail.com>

RUN mkdir /app
WORKDIR /app

ADD Gemfile /app/Gemfile
ADD Gemfile.lock /app/Gemfile.lock

ENV RAILS_ENV production
RUN bundle install --deployment

ADD . /app
RUN bundle exec rake assets:precompile RAILS_ENV=production


RUN apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
```

Then the nginx container I build it with this Dockerfile

```
FROM nginx:1.7.9
MAINTAINER JP Colomer <jpcolomer@gmail.com>

RUN mkdir -p /etc/nginx/ssl

ADD /path/to/app.crt /etc/nginx/ssl/app.crt
ADD /path/to/app.key /etc/nginx/ssl/app.key

ADD app.conf /etc/nginx/sites-available/app.conf
RUN ln -s /etc/nginx/sites-available/app.conf /etc/nginx/conf.d/app.conf
```

And use the following conf file for nginx.

```
upstream myapp {
  server web1:3000;
}

server {
  listen 443 default ssl;
  server_name myapp.com;

  ssl on;
  ssl_certificate /etc/nginx/ssl/app.crt;
  ssl_certificate_key /etc/nginx/ssl/app.key;

  location / {
    proxy_set_header Host $http_host;
    proxy_set_header  X-Real-IP      $remote_addr; # pass on real client's IP
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto https;
    proxy_pass http://myapp; # match the name of upstream directive which is defined above
  }
}
```

Finally, it's important to run rails container first and then nginx container.

```
docker run --name app -d my_app bundle exec rails server
docker run --link app:web1 -p 443:443 -d my_nginx
```
