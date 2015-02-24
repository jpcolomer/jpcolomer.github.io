---
layout:     post
title:      Rails app Zero Downtime deployment with Docker
date:       2015-02-24 13:09:00
categories: docker
---

Following my last post. I've been trying to do a zero downtime
deployment of my rails app.

I've had to change my nginx configuration. And I've written a bash
script to update my containers.

```
upstream newadmin {
  server dockerhost:8080;
  server dockerhost:8081;
}

server {
  listen 80;
  server_name myapp.com;
  server_tokens off;
  return 301 https://$server_name$request_uri;
}

server {
  listen 443 default ssl;
  server_name myapp.com;

  ssl on;
  ssl_certificate /etc/nginx/ssl/app.crt;
  ssl_certificate_key /etc/nginx/ssl/app.key;

  location / {
    proxy_set_header  X-Real-IP      $remote_addr; # pass on real client's IP
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto https;
    proxy_set_header Host $http_host;
    proxy_redirect off;
    proxy_pass http://myapp; # match the name of upstream directive which is defined above
  }

  location ~* ^/assets/ {
    # Per RFC2616 - 1 year maximum expiry
    expires 1y;
    add_header Cache-Control public;

    # Some browsers still send conditional-GET requests if there's a
    # Last-Modified header or an ETag header even if they haven't
    # reached the expiry date sent in the Expires header.
    add_header Last-Modified "";
    add_header ETag "";
    break;
  }
}
```

On this case, 2 containers of the app are runned and they publish their 3000 port to their host's ports 8080 and 8081.

Finally, here is the bash script I've written

{% highlight bash %}
#!/bin/bash

IMAGE="docker-registry.myregistry.com/myapp:$1"
NAME1=myapp1
NAME2=myapp2

NGINX_IMAGE="docker-registry.myregistry.com/my_nginx:$2"
NGINX_NAME=my_nginx

echo "Stop $NAME1"
docker stop $NAME1 > /dev/null
docker rm $NAME1 > /dev/null

echo "Start $NAME1, version $1"

while [[ -z "$(docker ps | grep $NAME1 |  awk '{print $NF}')"  ]]; do
  docker run \
    -p 8080:3000 \
    --restart always \
    --name $NAME1 \
    -d $IMAGE bundle exec rails server > /dev/null
done

echo -ne "Waiting $NAME1 "
until curl -s http://localhost:8080 > /dev/null; do
  echo -ne "."
  sleep 0.5
done
echo "."

echo "Stop $NAME2"
docker stop $NAME2 > /dev/null
docker rm $NAME2 > /dev/null

echo "Start $NAME2, version $1"

while [[ -z "$(docker ps | grep $NAME2 |  awk '{print $NF}')"  ]]; do
  docker run \
    -p 8081:3000 \
    --restart always \
    --name $NAME2 \
    -d $IMAGE bundle exec rails server > /dev/null
done

if [ ! -z "$2" ]
  then
    echo "Stop $NGINX_NAME"
    docker stop $NGINX_NAME > /dev/null
    docker rm $NGINX_NAME > /dev/null

    echo "START $NGINX_NAME, version $2"

    while [[ -z "$(docker ps | grep $NGINX_NAME |  awk '{print $NF}')"  ]]; do
      docker run \
        -p 443:443 \
        -p 80:80 \
        --name $NGINX_NAME \
        --add-host=dockerhost:$(ip route | awk '/docker0/ { print $NF }') \
        -v ~/myapp.conf:/etc/nginx/sites-available/myapp.conf \
        -d $NGINX_IMAGE > /dev/null
    done
fi
{% endhighlight %}

Finally, to update myapps and nginx

```
sudo bash my_script.sh myappLatestVersion nginxLatestVersion
```

If you want only to update myapp's containers

```
sudo bash my_script.sh myappLatestVersion
```

First, this script stops one container, then it starts that container with a new image and waits until Rails answers to a GET request. Then it stops and updates the second container.
It also adds a host, dockerhost, to be able to use it in the nginx conf file. Docker containers are runned inside a loop because I've been having a random problem when docker containers are stopped and started afterwards.
