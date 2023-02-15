---
layout:     post
show_meta: true
title:      CSRF in Sinatra with Padrino
header:     CSRF in Sinatra with Padrino
date:       2016-05-14 10:00:00
summary:  Enable CSRF for form submissions in Sinatra with Padrino form helpers
categories: ruby sinatra padrino
author: Chee Yeo
---

In a recent sinatra app I'm working on I require the use of dynamic form helpers to generate html forms similar to rails.

After several attempts at trying out various options, I settled on Padrino
as it has features similar to Rails and are well tested.

However to enable `authenticity_token` within the forms, one has to jump
over several hoops to activate it properly.

#### Enable sessions and protect_from_csrf

You need to enable the following settings in Sinatra, especially
sessions and session_secret since the `authenticity_token` in stored
within the session hash:

{% highlight ruby linenos %}
enable :sessions
set :session_secret, SecureRandom.hex(32)
set :protect_from_csrf, true
{% endhighlight %}

The `protect_from_csrf` is required else it will fail with __'protect_from_csrf not found'__. Even if you don't need to use the `authenticity_token` feature, you can disable it completely (not recommended):

{% highlight ruby linenos %}
disable :protect_from_csrf
{% endhighlight %}

#### Enable Rack Middleware

Secondly, you need to activate the Rack middleware with 'use' to actually
enable CSRF checking. Padrino uses either Rack::Protection::AuthenticityToken
or Padrino::AuthenticityToken based on whether you need to filter out certain routes. This can be seen in the following source code:

[https://github.com/padrino/padrino-framework/blob/master/padrino-core/lib/padrino-core/application/application_setup.rb#L165](https://github.com/padrino/padrino-framework/blob/master/padrino-core/lib/padrino-core/application/application_setup.rb#L165){:target="_blank"}

{% highlight ruby linenos %}
builder.use(options[:except] ? Padrino::AuthenticityToken : Rack::Protection::AuthenticityToken, options)
{% endhighlight %}

Since I'm using only the Padrino helpers in a Sinatra app, I need to include
the middleware. For a Padrino application, the builder will do the below
for you automatically based on the settings above.

If you are allowing all routes to be checked for CSRF, you can just enable the following:

{% highlight ruby linenos %}
use Rack::Protection::AuthenticityToken
{% endhighlight %}

If you have API endpoints which do not require CSRF checking as there
are no form submissions:

{% highlight ruby linenos %}
use Padrino::AuthenticityToken, :except => ["/api"]
{% endhighlight %}

The difference being `Padrino::AuthenticityToken` subclasses `Rack::Protection::AuthenticityToken` but in its call block it actually
checks to see if the `:except` option is set and exclude those routes specified. The other routes get passed through to `Rack::Protection::AuthenticityToken` in its super call.

This can be seen in the middleware itself:

[https://github.com/padrino/padrino-framework/blob/master/padrino-core/lib/padrino-core/application/authenticity_token.rb#L10-L24](https://github.com/padrino/padrino-framework/blob/master/padrino-core/lib/padrino-core/application/authenticity_token.rb#L10-L24){:target => "blank"}

{% highlight ruby linenos %}
def call(env)
  if except?(env)
    @app.call(env)
  else
    super
  end
end

def except?(env)
  return false unless @except
  path_info = env['PATH_INFO']
  @except.is_a?(Proc) ? @except.call(env) : @except.any?{|path|
    path.is_a?(Regexp) ? path.match(path_info) : path == path_info }
end
{% endhighlight %}

I have to require `"padrino-core/application/authenticity_token"` manually in addition to `'padrino-helpers'` but due to the modularity of Padrino, I only need to include what I need in my use case.

## A complete config

The complete config is as so:

{% highlight ruby linenos %}
require "padrino-helpers"
require "padrino-core/application/authenticity_token"

class MyApp < Sinatra::Base
  configure do
    enable :sessions
    set :session_secret, SecureRandom.hex(32)
    # enable authenticity_token in forms
    set :protect_from_csrf, true
    # actual checks for csrf tokens from form submissions
    use Padrino::AuthenticityToken, :except => ["/api"]
  end
end
{% endhighlight %}

Happy Hacking!
