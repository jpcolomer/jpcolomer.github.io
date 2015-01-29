---
layout:     post
title:      How to debug a rails app using docker and fig
date:       2015-01-29 13:09:00
categories: docker rails
---

If you've tried rails with docker you should definitely check fig. It's
sort of a Docker orchestration and dependency manager.

### How it works?
Define a `fig.yml` with your containers and variables

```
db:
  image: postgres
  ports:
    - "5432"
redis:
  image: redis
  ports:
    - "6379"
web:
  image: my_own_app_image
  command: bundle exec rails s
  links:
    - db
    - redis
  ports:
    - "3000:3000"
```

If this is the first time you run your app, you should migrate your db.

```
fig run web rake db:migrate
```

Then to run your app:

```
fig up
```

One of the problems I've been having is debugging with pry or debugger while runing my rails
app with fig.

I found an issue in github (https://github.com/docker/fig/issues/359).
Basically, with that feature now is possible to interactively debug your
rails app.
You could wait until fig 1.1.0 is realased, or install fig from github's master
branch.

First step, clone fig's github

```
git clone https://github.com/docker/fig.git
```

Next step, install fig
```
cd fig
python setup.py install
```

Finally, it's possible to run the rails app and interactively debug. In
this case you should execute the following command.

```
fig run --service-ports web bundle exec rails s
```
