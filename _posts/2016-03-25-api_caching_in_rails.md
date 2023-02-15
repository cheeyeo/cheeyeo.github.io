---
layout:     post
show_meta: true
title:      Rails API Caching
header:     Rails API Caching
date:       2016-03-25 21:00:00
summary:  Caching in Rails API and pitfalls to avoid
categories: ruby rails api
author: Chee Yeo
---

I have been working on implementing HTTP Caching at work in a Rails project and thought I would share some of the 'gotchas' I encountered along the way.

In Rails, we can implement HTTP caching like this:

{% highlight ruby linenos %}
def index
  @posts = Post.all

  if stale?(etag: @posts, last_modified: @posts.maximum(:updated_at))
    render json: @posts
  end
end
{% endhighlight %}

Using the above example, the first client call will receive a 200 response back and JSON is rendered. The second call will return a 304 not modified response and the response body will be blank. If the client is a web browser it will serve the cached response.

Below are a list of issues I encountered when implementing HTTP caching and how to resolve them.

## 1. Use stale?

We can only utilize __stale?__ as above when rendering JSON responses. __fresh_when__ works only in the context of a normal render flow of a controller i.e. if the client cache is not fresh, then we re-render the view. It does not accept a block like __stale?__ does and hence not applicable for calls such as __'render json:'__


## 2. Testing 304 responses

In integration / controller tests, we are making a direct request to the api endpoint. This means that to test for an effective 304 response we need to test for empty responses and set the __@request last_modified date__ accordingly:

{% highlight ruby linenos %}
# some controller / integration test

test "Returns 304 response" do
  @request.last_modified = @posts.maximum(:updated_at).httpdate
  get "/api/posts"
  assert @response.body.blank?
  assert_equal 304, @response.status.to_i
end
{% endhighlight %}

## 3. Disable Rack::MiniProfiler caching

Rack::MiniProfiler middleware will remove headers related to caching and as such, stale? will ALWAYS RETURN TRUE. We can disable caching altogether using the following initializer:

{% highlight ruby linenos %}
Rack::MiniProfiler.config.disable_caching = false
{% endhighlight %}

## 4. Enable 'IfModified' in jquery $.ajax

To make ajax calls to an api with http conditionals, we need to set __'cache'__ to true
and __'ifModified'__ to true in the $.ajax options block.

Setting __'cache'__ to false will result requests being appended with an extra ___=timestamp__ parameter, which will flag up __'unpermitted parameter'__ error when used with strong parameters.

Setting __'ifModified'__ to true will allow the client to check for the __'Last-Modified'__ header and serve the cached response if its a 304 response.

{% highlight javascript linenos %}
$.ajax({
    type : 'GET',
    url : 'posts',
    cache: true,
    ifModified: true,
    ....

})
{% endhighlight %}

## 5. Expose headers in Rack::Cors

If we are also using rack-cors with a javascript client, we also need to also expose the caching headers else the javascript will not receive the headers and will keep making 200 requests.

{% highlight ruby linenos %}
Rails.application.config.middleware.insert_before 0, "Rack::Cors" do
  allow do
    origins '*'
    resource '*',
    headers: :any,
    methods: [:get, :post, :options, :patch, :delete],
    expose: ['ETag', 'If-Last-Modified', 'If-None-Match', 'Last-Modified']
  end
end
{% endhighlight %}

There is another gotcha which has to do with __'Unpermitted parameter: format'__ which has more to do with the way strong parameters work in later versions of Rails which is outside the scope of this article and will be presented in a later article.

Lastly, always implement some kind of caching mechanism, especially for API endpoints which renders structured data such as JSON from ActiveRecord models which may have associations between them.

When used with ActiveRecord::Serializers and its built-in key-based caching, we can cache individual items similar to a russian-doll approach followed by the entire response using HTTP caching, resulting in better overall performance.
