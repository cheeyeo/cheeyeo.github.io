---
layout:     post
show_meta: true
title:      Rails 4.2 ActionMailer
header:     Rails 4.2 ActionMailer
date:       2015-07-22 21:55:00
summary:  Using ActionMailer in Rails 4.2 wth sidekiq
categories: rails actionmailer activejob sidekiq
author: Chee Yeo
---

From Rails 4.2, you can deliver an email asynchronously using ActiveJob which is built into ActionMailer.

ActionMailer now comes with two additional methods:

{% highlight ruby linenos %}
deliver_later
deliver_now
{% endhighlight %}

__deliver_now__ is similar to the original deliver method; it sends the email synchronously.

__deliver_later__ makes use of your background job system to send the email.


Note that there is a difference between the bang and non-bang methods.

With deliver_later! and its bang methods, no exceptions will be raised if an error occurs when sending the mail. Without the bang methods, exceptions will be raised if the mail fails to send.

To get this to work properly, some additional changes need to be made.

Firstly, make sure that your background job system is setup and working properly. We are using Sidekiq for this example.

This assumes that you have __config.active_job.queue_adapter__ set like so:

{% highlight ruby linenos %}
# within config/application.rb

config.active_job.queue_adapter = :sidekiq
{% endhighlight %}

Next you need to add the __'mailers'__ queue into the config file. ActionMailer by default places background deliveries into this queue.

For sidekiq:

{% highlight ruby linenos %}
# config/sidekiq.yml

:queues:
  - [mailers, 1]
{% endhighlight %}

Create a mailer and call deliver_later and if it all goes well you should see the below in your log file:

{% highlight ruby linenos %}
[ActiveJob] [ActionMailer::DeliveryJob] [fd25b472-9d80-47f2-9ea6-5da0e3cf7552] Performed ActionMailer::DeliveryJob from Sidekiq(mailers) in 3160.77ms
{% endhighlight %}

# Tracking exceptions

It is inevitable that errors might occur in your mailers. When using sidekiq, it has a default mechanism for queuing and retrying failed jobs. If an exception occurs in your mailer when sending asynchronously, it will be placed into this queue and retried for at least 25 times as a default.

To be notified of when an exception occurs, we can hook into sidekiq eror handlers like so:

{% highlight ruby linenos %}

# config/initializers/sidekiq.rb
Sidekiq.configure_server do |config|
  config.error_handlers << Proc.new {|ex,ctx_hash|
    SidekiqError.log_exception(ex, ctx_hash)
  }
end

# sidekiq_error.rb

class SidekiqError
  def self.logger
    @logger ||= Logger.new("#{Rails.root}/log/sidekiq.log")
  end

  def self.log_exception(e, args={})
    self.logger.error "Error class: #{args["error_class"]}"
    self.logger.error "Error message: #{args["error_message"]}"
    self.logger.error e.message
    st = e.backtrace.join("\n")

    self.logger.error st
    self.logger.error "Failed At: #{args["failed_at"]}"
    self.logger.error "Retry Count: #{args["retry_count"]}"
    self.logger.error "Retried At: #{args["retried_at"]}"
  end
end

{% endhighlight %}

I still have not worked out a way to rescue exceptions within async mailers as the class is defined automatically in ActionMailer when you call deliver_later from your mailer:

{% highlight ruby linenos %}
# actionmailer/lib/action_mailer/delivery_job.rb

require 'active_job'

module ActionMailer
  # The <tt>ActionMailer::DeliveryJob</tt> class is used when you
  # want to send emails outside of the request-response cycle.
  class DeliveryJob < ActiveJob::Base # :nodoc:
    queue_as { ActionMailer::Base.deliver_later_queue_name }

    def perform(mailer, mail_method, delivery_method, *args) #:nodoc:
      mailer.constantize.public_send(mail_method, *args).send(delivery_method)
    end
  end
end

{% endhighlight %}

One approach would be to create a refinement for ActionMailer::MessageDelivery enqueue_delivery private method to use a custom ActiveJob object which has a rescue_from block but that is the topic of another blog post.

Happy Hacking!
