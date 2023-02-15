---
layout:     post
show_meta: true
title:      Setting up SSL with Jekyll using LetsEncrypt
header:     Setting up SSL with Jekyll using LetsEncrypt
date:       2016-02-07 21:00:00
summary:  Using LetsEncrypt to generate SSL certs for use with a Jekyll blog.
categories: letsencrypt ruby jekyll ssl
author: Chee Yeo
---

[LetsEncrypt](https://letsencrypt.org/){:target="_blank"} is a CA which allows you to register for free SSL certificates for use on your website. The following are the steps I took to register for a SSL certificate for my current website.

## Step 1: Getting letsencrypt
The first step is to clone the letsencrypt repo from github. It includes the client which you will need to generate the certificates:

{% highlight console linenos %}
cd letsencrypt

sudo ./letsencrypt-auto certonly -a manual --rsa-key-size 4096 -d mysite.com -d www.mysite.com
{% endhighlight %}

The above command essentially reads to generate SSL for my domains cheeyeo.uk and www.cheeyeo.uk with a rsa key size of 4096. The certonly flag means to use only the "standalone" or "webroot" plugins to request for a certificate. The letsencrypt client can integrate with Apache and Nginx servers directly but that is outside the scope of the current article.

The sudo flag is required as it writes to certain system directories such as /etc/letsencrypt. Without it, I kept receiving python missing module errors and running as root seems to resolve it.

Once the above completes, you will be prompted with a screen asking you for permission to log your IP address.


Once confirmed, you will be presented with instructions on creating a specific url on your server that needs to return a nounce or token.

This is usually in the form of:

{% highlight console linenos %}
Make sure your web server displays the following content at
http://www.yourdomain.com/.well-known/acme-challenge/weoEFKS-aasksdSKCKEIFIXCNKSKQwa3d35ds30_sDKIS before continuing:

weoEFKS-aasksdSKCKEIFIXCNKSKQwa3d35ds30_sDKIS.Rso39djaklj3sdlkjckxmsne3a
Content-Type header MUST be set to text/plain.
{% endhighlight %}

(The above is just an example and you will see a different output!)

For Jekyll, since we are serving static resources from the _site directory, this means that the folder will always be wiped on each deploy.

To workaround this, I added the __keep_files__ directive to my _config.yml file:

{% highlight yaml linenos %}
# _config.yml
....

keep_files: [
  '.well-known/acme-challenge'
]

....
{% endhighlight %}

Then we create the directories and the required file:

{% highlight console linenos %}
cd _site/.well-known/acme-challenge

echo -n "weoEFKS-aasksdSKCKEIFIXCNKSKQwa3d35ds30_sDKIS.Rso39djaklj3sdlkjckxmsne3a" > "weoEFKS-aasksdSKCKEIFIXCNKSKQwa3d35ds30_sDKIS"
{% endhighlight %}

Next, I added the ___site/.well-known__ subdirectory into a git commit, ready to be pushed to the Heroku server. Once that is done, check that you can access it manuall by visiting the url above directly.

Please also note that you need to repeat the process above for each domain you listed in the -d flag. For example, I needed to do the deploy twice because I added 2 domains: 'mysite.com' and 'www.mysite.com'.

Assuming you don't run into any glitches, you have completed step 1 of the process.

Your certificates will be downloaded and stored on your local filesystem under __/etc/letsencrypt/live/mysite.com__. There should be 4 files namely: __cert.pem; chain.pem; fullchain.pem; privkey.pem__

I copied them into my jekyll folder and gitignore it as we need them for step 2.

## Step 2: Setting up SSL on Heroku

You need to add the SSL addon to use SSL and add the certs from step 1:

{% highlight console linenos %}

heroku addons:create ssl:endpoint

heroku certs:add ssl/cert.pem ssl/privkey.pem ssl/chain.pem ssl/fullchain.pem
{% endhighlight %}

To check that its all setup properly:
{% highlight console linenos %}
heroku certs

heroku certs:info
{% endhighlight %}

If it all goes well, you should see a dedicated sslenpoint with an ending domain of __herokussl.com__ which you need for step 3. Also, check that the heroku certs:info return __Trusted__ field as __True__ before proceeding further.

## Step 3: Configuring DNS

Next we need to update the CNAME record of your DNS settings to point to the new heroku ssl endpoint above. In my case it took several hours for the changes to propagate.

Once done you should see something like the following:

{% highlight console linenos %}
host www.mysite.com

mysite.herokussl.com is an alias for xxxxxxxx
{% endhighlight %}

To check that the ssl is successfully deployed, run the following:

{% highlight console linenos %}

curl -kvI www.mysite.com

* Connected to www.mysite.com (XXXXXX) port 443 (#0)
* TLS 1.2 connection using TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
* Server certificate: mysite.com
* Server certificate: Let's Encrypt Authority X1
* Server certificate: DST Root CA X3
{% endhighlight %}

## Step 4: Jekyll and heroku specific issues

The above setup works for me up until the point when I started getting Mixed content warnings in the chrome browser when visiting individual posts pages. Make sure that if you are using the __site.baseurl__ call to generate the post url in _config.yml, update it to your full https version.

The other issue is redirection from non-ssl to ssl links. Since I'm only serving static pages using Rack, links to the non-ssl pages still renders the non-ssl versions of the site.

To fix this, I added a 301 permanent redirect from my DNS console.

That only managed to fix the issue of root request of __mysite.com__ to be redirected to __https://www.mysite.com__. Calls to __www.mysite.com__ still reaches the non-ssl version.

To fix this, I used the [rack-ssl-enforcer](https://github.com/tobmatth/rack-ssl-enforcer){:target="_blank"} gem. By requiring it into the config.ru file and redeploying it, all requests for the root domain or non-ssl gets redirected to the ssl version of the site.

Other recommendations from other users with similar issues as mine include using an nginx buildpack to serve as a proxy but that seems a lot of work in my use case. Perhaps as another blog post in the future.

Success at last!

## Conclusion

LetsEncrypt is great and I would defintely suggest taking it for a test run. Note that the certificates are only valid for 3 months maximum. This is a security feature and you can [read more about it here](https://letsencrypt.org/2015/11/09/why-90-days.html){:target="_blank"}.

For a more detailed read on setting it up on OS X, [read about it here](https://community.letsencrypt.org/t/tutorial-for-os-x-local-certificates-and-shared-hosting/6859){:target="_blank"}
