---
layout:     post
title:      Bunder local and remote path
date:       2015-01-20 13:09:00
summary:    Tell bundler to use a local copy of a gem repository
categories: ruby
---

When you want to use a private gem that is developed locally and also used in one of my projects
bundler complains because there are two sources for the same gem.

{% highlight ruby %}
gem 'my_gem', path: '~/Projects/gems/my_gem', group: :development
gem 'my_gem', github: 'jpcolomer/my_gem', branch: 'master', group: :production
{% endhighlight %}

What you could do is to tell bundler to use the local copy of this gem:

```
bundle config local.my_gem ~/Projects/gems/my_gem
```

Then you can clear your Gemfile and leave the remote path:

{% highlight ruby %}
gem 'my_gem', github: 'jpcolomer/my_gem', branch: 'master'
{% endhighlight %}

Finally, you need to disable local branch check in bundler:

```
bundle config disable_local_branch_check true
```

Otherwise bundle will fail if ~/Projects/gems/my_gem isn't using master.
