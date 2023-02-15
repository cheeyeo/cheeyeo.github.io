---
layout:     post
show_meta: true
title:      ETags and http caching in Rails 5
header:     ETags and http caching in Rails 5
date:       2016-08-09 00:00:00
summary: ETags and http caching in Rails 5
categories: ruby rails5 cache http
author: Chee Yeo
---

Rails 5 has introduced some changes to the way HTTP caching works.

ETags are by default now set to `weak` mode in Rails 5. This is because Rails does not do byte-to-byte checking for equality which is comparing changes in both the headers and response body.

Weak Etags only compares the response body for differences, which is also the default behaviour in earlier versions of Rails.

Assuming we have a Users controller and an expensive rendering action on the index action which we want to call render only if the users have been updated.

We can implement a conditional get using `stale?` like so:

{% highlight ruby linenos %}
if stale?(etag: @users, last_modified: @users.maximum(:updated_at)
  respond_to do |format|
    format.html
  end
end
{% endhighlight %}

Assuming we made a curl request to the endpoint /users:

{% highlight bash linenos %}
curl -i http://localhost:3000/users
{% endhighlight %}

It will return a 200 response:

{% highlight bash linenos %}
HTTP/1.1 200 OK
X-Frame-Options: SAMEORIGIN
X-XSS-Protection: 1; mode=block
X-Content-Type-Options: nosniff
ETag: W/"8e49d1a448037c58d93ea3a00d6edf87"
{% endhighlight %}

Note that the ETag now has a `W/` prefix to indicate its a weak etag, which is the default in Rails 5.

If we now use the ETag value and make a second curl request with the header `If-None-Match` set to the same ETag value we should get a 304 response.

{% highlight bash linenos %}
curl -i -H 'If-None-Match: "8e49d1a448037c58d93ea3a00d6edf87"' http://localhost:3000/users
{% endhighlight %}

However, the above will keep returning 200 as it is not matching the response. You need to specify the entire ETag value like so:

{% highlight bash linenos %}
curl -i -H 'If-None-Match: W/"8e49d1a448037c58d93ea3a00d6edf87"' http://localhost:3000/users
{% endhighlight %}

To use strong etags (without the W/ prefix), you can change the `etag` parameter in the `stale?` method:

{% highlight ruby linenos %}
if stale?(strong_etag: @users, last_modified: @users.maximum(:updated_at))
  respond_to do |format|
    format.html
  end
end
{% endhighlight %}

Stay curious and keep hacking!
