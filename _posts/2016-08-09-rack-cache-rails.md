---
layout:     post
show_meta: true
title:      Rack::Cache in Rails 5
header:     Rack::Cache in Rails 5
date:       2016-08-09 00:00:00
summary: Setting up and using Rack::Cache in Rails 5.
categories: ruby rails5 cache http
author: Chee Yeo
---

Rack::Cache is a middleware which acts as a proxy cache between your client and the app. Initial requests are sent directly to the app, which then gets cached by the proxy. Subsequent requests go only to the proxy and bypass the app completely. It is used to cache assets that don't change often but can also be used to cache http responses.

### Setting up caching in Rails 5.

You need to have `gem rack-cache` specified in the Gemfile.

Then, run the following task:

{% highlight ruby linenos %}
bin/rails dev:caching
{% endhighlight %}

This will create an empty file `tmp/caching-dev.txt`.

Within `config/development.rb`, you will see the following code:

{% highlight ruby linenos %}
# Enable/disable caching. By default caching is disabled.
  if Rails.root.join('tmp/caching-dev.txt').exist?
    config.action_controller.perform_caching = true
    # set below to true to use rack_cache
    # also make sure rack-cache gem in gemfile
    config.action_dispatch.rack_cache = {
      verbose: true,
      metastore:   'file:/appdir/tmp',
      entitystore: 'file:/appdir/tmp',
      allow_reload: true
    }

    config.cache_store = :memory_store
    config.public_file_server.headers = {
      'Cache-Control' => 'public, max-age=172800'
    }

  else
    config.action_controller.perform_caching = false
    config.cache_store = :null_store
  end
{% endhighlight %}

I enabled `Rack::Cache` by setting the `config.action_dispatch.rack_cache` key and passing in a hash of custom options. It can also be set to `true` and it will use the defaults defined in the Rails middleware stack.

I have also set `verbose` to true in development mode so we can check if it is working.

Run `bin/rails middleware` and you should see the following:

{% highlight ruby linenos %}
use Rack::Sendfile
use ActionDispatch::Static
use Module
use ActionDispatch::Executor
{% endhighlight %}

The presence of `Module` under `ActionDispatch::Static` means `Rack::Cache` is loaded. You can check the [Rails default middleware stack]{:target => '_blank'} to confirm the ordering.

### Workings of Rack::Cache

Underneath the hood, Rack::Cache processes each request through the middleware stack and compares certain http request headers to determine if it should fetch the resource from the cache or to forward it to the application.

There are certain conditions under which requests will bypass the cache and goes completely to the app:

* A non-GET request is made.

* `allow_reload` is set to `true` in the configuration and client sends a `Cache-Control: no-cache` header


* `allow_revalidate` is set to `true` in the configuration and client sends a `Cache-Control: max-age=0` header

* if the header contains authorization fields such as `Authorization` or 'Cookie', in which case Rack::Cache considers it to be private and will not cache it.

This means that if you are using http conditionals methods in your controller actions such as `stale?` method, it might lead to surprising behaviour.

Given an example application with a Users controller which has a conditional:

{% highlight ruby linenos %}
class UsersController < ApplicationController
  def index
    @users = User.all

    if stale?(etag: @users, last_modified: @users.maximum(:updated_at), public: true)
      respond_to do |format|
        format.html
      end
    end
  end
end
{% endhighlight %}

The option `public` must be set to `true` to allow Rack::Cache to store the content.

The first request will bypass `Rack::Cache`, hits the application and returns a 200 response but also the additional headers:

{% highlight bash linenos %}
curl -i http://localhost:3000/users

# Response Headers
# ...
X-Rack-Cache: miss, ignore, store
{% endhighlight %}

{% highlight ruby linenos %}
Started GET "/users" for ::1 at 2016-08-09 16:58:36 +0100
  ActiveRecord::SchemaMigration Load (0.3ms)  SELECT `schema_migrations`.* FROM `schema_migrations`
Processing by UsersController#index as */*
   (0.4ms)  SELECT MAX(`users`.`updated_at`) FROM `users`
   (0.3ms)  SELECT COUNT(*) AS `size`, MAX(`users`.`updated_at`) AS timestamp FROM `users`
  Rendering users/index.html.erb within layouts/application
  User Load (0.3ms)  SELECT `users`.* FROM `users`
  Rendered users/index.html.erb within layouts/application (9.0ms)
Completed 200 OK in 347ms (Views: 321.2ms | ActiveRecord: 4.7ms)
{% endhighlight %}

`X-Rack-Cache: miss` means that the response is not found in the cache store so it is fetched from the application and then stored within the cache.

A further request shows the following:

{% highlight bash linenos %}
# request
curl -i http://localhost:3000/users

# part of the response headers
# ....
X-Rack-Cache: stale, valid, store
{% endhighlight %}

{% highlight ruby linenos %}
Started GET "/users" for ::1 at 2016-08-09 16:59:56 +0100
Processing by UsersController#index as */*
   (0.4ms)  SELECT MAX(`users`.`updated_at`) FROM `users`
   (0.3ms)  SELECT COUNT(*) AS `size`, MAX(`users`.`updated_at`) AS timestamp FROM `users`
Completed 304 Not Modified in 3ms (ActiveRecord: 0.7ms)
{% endhighlight %}

Note that the application is now returning a 304 response and no rendering has occured on the application side but we still receive a HTML response in the terminal.

This is because `Rack::Cache` has stored the initial first request and is returning it as the response, bypassing the application completely.

The `X-Rack-Cache` header indicates that there is a cache hit and it is rendering the content directly from the cache store, which is specified in the `entitystore` configuration directory.

If you look within this directory, you will see some folders. The `X-Content-Digest` shows the location of this cached response: the first 2 digits denote the directory and the remaining characters are the filename.

In our example above, we have:

{% highlight bash linenos %}
# <11> directory
# <463e97d33861d8ca81e9507e1d8d2e85cf2368> filename

X-Content-Digest: 11463e97d33861d8ca81e9507e1d8d2e85cf2368
{% endhighlight %}

which means the cached html fragment is stored in `tmp/11/463e97d33861d8ca81e9507e1d8d2e85cf2368`. Opening it should show the entire html response.

By sending an `ETag` or `Last-Modified` header value that matches, we will not receive any html content back as no rendering has taken place, which is similar to just having http caching on its own.

One point of note is that Rails 5 use `weak etags` by default, which means you would need to change the curl syntax to get it to work (note the W/ prefix):

{% highlight bash linenos %}
curl -i -H 'If-None-Match: W/"e2b3d25cf8426f3cc00dcd43f8ac2148"' http://localhost:3000/users
{% endhighlight %}

If you have `allow_reload` or `allow_revalidate` set, you can always bypass the cache, which is useful for testing:

{% highlight bash linenos %}
curl -i -H 'Cache-Control: no-cache' http://localhost:3000/users
{% endhighlight %}

This will cause the rendering to occur and invalidate the cache.

In Rails 5, ActiveRecord relation objects now return a cache key of the following format:

{% highlight bash linenos %}
<class of records>/query-<md5 hash>-<nos of records>-<timestamp of most updated record in collection>

"users/query-b37955d0f26d583466428665d31ecd71-3-20160809153910000000"
{% endhighlight %}

In our UsersController, since we are passing `@users` to the `etag` parameter, Rails will automatically use the above to calculate the cache key.

Even if the html template were to be updated, Rack::Cache will keep rendering the cached version.

Only when a single user in the returned relation is updated will it render the new template changes again since the cache key will be invalidated.

This might cause some confusion in development mode but can be easily bypassed by updating the `updated_at` field of any user in a console:

{% highlight ruby linenos %}
  >> User.last.touch
{% endhighlight %}

Subsequent request will now render the updated content:

{% highlight bash linenos %}
curl -i http://localhost:3000/users

# renders the updated content and storing it in the cache
X-Rack-Cache: stale, invalid, ignore, store
{% endhighlight %}

The `X-Rack-Cache: stale, invalid, ignore, store` indicates that the updated content has been fetched and stored in the cache. Further request will return `X-Rack-Cache: stale, valid, store`, with a status code of 304 from the application and the cached content from the proxy.

I hope this has helped in understanding and using `Rack::Cache` in Rails 5.

Stay curious and keep hacking!

[Rails default middleware stack]: https://github.com/rails/rails/blob/5-0-stable/railties/lib/rails/application/default_middleware_stack.rb


### Further information
- [rack-cache github source](https://github.com/rtomayko/rack-cache)
- [rack cache configuration options](https://github.com/rtomayko/rack-cache/blob/master/doc/configuration.markdown)
