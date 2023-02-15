---
layout:     post
show_meta: true
title:      Auto renewal letsencrypt certs on CentOS 7
header:     Auto renewal letsencrypt certs on CentOS 7
date:       2018-03-23 00:00:00
summary:  How to setup auto renewal for letsencrypt on CentOS 7
categories: linux certbot devops
author: Chee Yeo
---

In a previous article, I documented how to setup certbot on a centos 7 server. Since all letsencrypt certs have a limited lifespan of 3 months, its important to understand and be able to automate the renewal of the certs using certbot without any downtime if possible.

Certbot has built-in plugins for common server types such as nginx and apache to aid with the renewal process. Assuming you followed the setup instructions on the letsencrypt setup docs for centos, you would have installed `certbot-nginx` using `sudo yum install certbot-nginx`.

There are a few problems I have with the above approach:

* The purpose or role of certbot is to obtain and renew the certs. Giving it control over your server process and configuration means increasing the attack surface vector in the event that there is a security issue with certbot.

* The plugin overwrites the server configs based on where the letsencrypt certs are stored. This presents a problem if you have custom configs and need to have custom paths for the certs.

* The plugin also links to its own server configs within your own configs which doesn't make sense. For example, in the certbot-nginx plugin above, it adds an additional line to the main ```/etc/nginx/nginx.conf``` file with an import to ```/etc/letsencrypt/le_tls_sni_01_cert_challenge.conf``` which contains another server block that imports certs from ```/var/lib/letsencrypt``` folder. This caused the nginx server to respond with an error when ```ssl_stapling``` is on as it referenced a cert which doesn't have the right CA, triggering a warning about invalid certs for ssl_stapling.

* Last but not least, when run together with certbot using the --nginx option, it fails with various different errors, even when running with an actual working version of certbot in virtualenv, which is baffling to work out.

For those reasons above and my sanity, I decided to look into how we can still automate the renewal process. Certbot accepts a webroot option which allows you to specify where the acme challenges can be stored. It also accepts a pre-hook argument which allows you to specify what happens after a successful renewal. Combining the two, this is what I come up with:

{% highlight console linenos %}
sudo ~/mycertbot/bin/certbot renew --webroot -w /home/web/site/.well-known/acme-challenge --post-hook "sudo systemctl restart nginx.service"
{% endhighlight %}

The renew command checks and renews the certs for multiple domains if it has expired. This is important as letsencrypt impose rate limiting on their apis. The above assumes that you have already created the webroot directory.

You would also need to add the corresponding entry into your server config to allow certbot to ping the path above. I added the below into my plain or port 80 nginx config server block:

{% highlight console linenos %}
location ^~ /.well-known/acme-challenge/ {
  default_type "text-plain";
  root /home/web/site/.well-known/acme-challenge;
}

{% endhighlight %}

Also, your server block for ssl config would need to be updated to point to where certbot stores the renewed certs. Assuming you have set it up, the first generated certs should be within ```/etc/letsencrypt/live/mydomain.com```. Your ssl server config would need to be updated to reflect the letsencrypt paths. Below is an example of my ssl config:

{% highlight console linenos %}
ssl_certificate /etc/letsencrypt/live/mydomain.com/fullchain.pem; # managed by Certbot

ssl_certificate_key /etc/letsencrypt/live/mydomain.com/privkey.pem; # managed by Certbot

ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

{% endhighlight %}

To setup the auto renewal process, we need to call certbot with the renew subcommand in a cron task which runs twice a day. The following is my crontab entry within /etc/crontab:

{% highlight console linenos %}
0 0,12 * * * root /home/web/mycertbot/bin/certbot --webroot -w /home/web/site/.well-known/acme-challenge --post-hook "sudo systemctl restart nginx.service" >> /home/web/certbot_renewal.log 2>&1
{% endhighlight %}

I redirected the output to a logfile so I can check for its status. I have also used the post-hook argument to restart nginx once the process completes successfully. The task is set to run twice a day at midnight and 12 noon.

Lastly, ensure that the system cron service is running:

{% highlight console linenos %}
sudo systemctl start crond.service

sudo systemctl status crond.service -l
{% endhighlight %}

You can also run the same commands above using the dry-run flag to test it out before you set it as a cron job.

I hope I have presented an approach for renewing the letsencrypt certs without having to use an external plugin to manage the server resource. Happy hacking

### Resources
- [Webroot plugin](https://certbot.eff.org/docs/using.html#webroot)

- [Certbot Managing Certificates](https://certbot.eff.org/docs/using.html#re-creating-and-updating-existing-certificates)

- [Pre and post validation hooks](https://certbot.eff.org/docs/using.html#pre-and-post-validation-hooks)
