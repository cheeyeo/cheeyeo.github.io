---
layout:     post
show_meta: true
title:      Rails 4.2 ActionMailer
header:     Rails 4.2 ActionMailer
date:       2015-01-31 15:56:00
summary:  Using ActionMailer in Rails 4.2
categories: rails actionmailer
author: Chee Yeo
---

From Rails 4.2, you are able to deliver an email via a background job.

ActionMailer classes come with two additional methods:

{% highlight ruby linenos %}
deliver_later
deliver_now
{% endhighlight %}

__deliver_now__ is similar to the original deliver method; it sends the email synchronously

__deliver_later__ makes use of your background job system to send the email.

To get this to work properly, some additional changes need to be made.

Firstly, make sure that your background job system is working properly.

This assumes that you have __config.active_job.queue_adapter__ set to something like sidekiq:

{% highlight ruby linenos %}
config.active_job.queue_adapter = :sidekiq
{% endhighlight %}

Next you need to add the __'mailers'__ queue into the config file.

ActionMailer by default places background deliveries into this queue.

In sidekiq:

{% highlight ruby linenos %}
# config/sidekiq.yml

:queues:
  - [mailers, 1]
{% endhighlight %}

Tail the development logs to make sure the deliveries are sent:

{% highlight ruby linenos %}
[ActiveJob] [ActionMailer::DeliveryJob] [fd25b472-9d80-47f2-9ea6-5da0e3cf7552] Performed ActionMailer::DeliveryJob from Sidekiq(mailers) in 3160.77ms
{% endhighlight %}

In a follow up post I will describe the steps needed to test mailers using deliver_later and the pitfalls to avoid.

Happy Hacking!
