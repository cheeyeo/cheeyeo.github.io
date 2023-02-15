---
layout:     post
show_meta: true
title:      Content Security Policy in Rails
header:     Content Security Policy in Rails
date:       2015-03-05 15:56:00
summary:  Using Content Security Policy in Rails
categories: ruby rails
author: Chee Yeo
---

[Content Security Policy](http://www.w3.org/TR/2015/CR-CSP2-20150219/){:target="_blank"} is a W3C specificaton that provides an additional layer of defence against XSS(Cross site scripting) and other content-based injection attacks by specifying a whitelist to restrict where a HTML page can load assets from.

It works by setting a HTTP header of Content-Security-Policy whereby one can define a whitelist of trusted sources of content which a browser will execute in the context of your application. Any untrusted sources will generate an error in the browser and will not be executed.

A good example of this is in the use of third party javascripts such as social media buttons. For example, we can define a simple rule to allow for the embed of google plus javascript in our webpages as follows:

{% highlight html %}
Content-Security-Policy: script-src 'self' https://apis.google.com
{% endhighlight %}

The rule above will only allow for javascripts from your local domain and that of apis.google.com to be loaded and executed in your origin. If you are building any application which allows for user generated content, this is an additional layer of security in addition to sanitizing the actual user data before rendering.

Since CSP is only a HTTP header we can actually implement it quite easily in say a before_action block in ApplicationController. I wanted to show how this can be done using the [secure_headers gem by Twitter](https://github.com/twitter/secureheaders){:target="_blank"} which allows for a more centralized and configurable approach.

First start by installing the gem in your application:

{% highlight ruby %}
gem 'secure_headers', :git => 'https://github.com/twitter/secureheaders', :branch => 'master', :require => 'secure_headers'
{% endhighlight %}

Then define an initializer say config/initializers/secure_headers.rb:

{% highlight ruby %}
::SecureHeaders::Configuration.configure do |config|
  config.csp = {
    :default_src => "https: self",
    :script_src => "https: self",
    :img_src => "self",
    :tag_report_uri => true,
    :enforce => true,
    :app_name => 'secure_headers_test',
    :report_uri => '/csp_reports'
  }
end

# and then simply include this in application_controller.rb
class ApplicationController < ActionController::Base
  ensure_security_headers
end
{% endhighlight %}

The rules in CSP are cascading. The __'default_src'__ key is the default fallback if any of the src attributes are not defined. For example, if we did not define an img_src attribute, it will load images with a __'https:'__ prefix regardless of its origin as well as images from your origin. The following images will be loaded:

{% highlight html %}
<img src="https://example.com/my_image.png" />
<img src="/assets/my_other_image.png" />
{% endhighlight %}

However, since we have defined __'img_src'__ to be only from __'self'__, only the second image will be loaded:

{% highlight html %}
<!-- throws an error in the chrome console of Refused to load the image 'https://example.com/my_image.png' because it violates the following Content Security Policy directive: "img-src 'self' data:".-->
<img src="https://example.com/my_image.png" />

<!-- this loads fine-->
<img src="/assets/my_other_image.png" />
{% endhighlight %}

In terms of inline scripts and styles, CSP will allow it if __'inline'__ is passed to the attribute definition. However, this will make it less secure than its intended purposes. It may require some refactoring to move some of the inline scripts into separate js files but this may not be possible all the time if you are using for example, content_for blocks to generate javascript on certain pages of your application.

secure_headers supports CSP Level 2 which allows for setting a __'nonce'__ to your script tags but will block everything else:

{% highlight ruby %}
:csp => {
  :default_src => 'self',
  :script_src => 'self nonce'
}

<script nonce="<%= @content_security_policy_nonce %>">
  console.log("whitelisted, so this will execute")
</script>

<script>
  console.log("not whitelisted, won't execute")
</script>
{% endhighlight %}

Another cool feature of CSP is its built in reporting functionality. By setting __'report_uri'__ and a url, CSP will automatically post a JSON request to the specified endpoint if any violation occurs. This is extremely useful in debugging CSP implementations.

Assuming we have a route '/csp_reports', we can gather the information like so:

{% highlight ruby %}

# within CspReports controller

class CspReportsController < ApplicationController
  def create
    if params["app_name"] == "SecureHeadersTest"
      data = request.body.read

      report = CspReport.new

      h = Oj.load(data)

      report.enforce = params["enforce"]
      report.app_name = params["app_name"]
      report.document_uri = h["csp-report"]["document-uri"]
      report.referrer = h["csp-report"]["referrer"]
      report.violated_directive = h["csp-report"]["violated-directive"]
      report.effective_directive = h["csp-report"]["effective-directive"]
      report.original_policy = h["csp-report"]["original-policy"]
      report.blocked_uri = h["csp-report"]["blocked-uri"]
      report.status_code = h["csp-report"]["status-code"]

      report.save!
    end

    head :ok
  end
end

{% endhighlight %}

An [example Rails 4.2 application](https://github.com/cheeyeo/Secure-headers-test-app){:target="_blank"} has been created to showcase the above.

Happy Hacking!

