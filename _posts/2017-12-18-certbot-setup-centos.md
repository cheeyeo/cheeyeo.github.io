---
layout:     post
show_meta: true
title:      Certbot setup on CentOS 7
header:     Certbot setup on CentOS 7
date:       2017-12-18 00:00:00
summary:  Self-help sanity guide on setting up certbot on centos7
categories: linux certbot devops
author: Chee Yeo
---

In order to learn more about devops, I took it upon myself a year ago to automate the process of setting up and publishing this __Jekyll__ blog to a cloud service. One of the things I wanted to achieve was to learn how to automate the process of generating SSL certificates using __certbot__.

It didn't dawn on me till much later that the whole idea of using __certbot__ is to automate the renewal process, which is when I got stuck.

If you follow the instructions on [the certbot website](https://certbot.eff.org/#centosrhel7-nginx){:target="_blank"}, it would install an older version of __pyOpenSSL__ using __yum__ through the __epel-release__ repo, which causes conflict errors such as __'Import OpenSSL not found'__ when the certbot binary is run. In addition, the python version is locked at 2.7.5 which is not updatable.

I realised that if I could install and run an up-to-date version of python, I could perhaps be able to install and run certbot through pip. After some research, I decided to install the latest version of python 2 which would work alongside the system version.

## Installing python

Below are the commands I ran to install python 2.7.14

{% highlight console linenos %}
# Only if you have installed certbot through yum previously
yum remove pyOpenSSL

# Needed later to build certbot
yum install openssl-devel python-devel

yum update

yum groupinstall -y "development tools"

# dependencies for building python
yum install -y zlib-devel bzip2-devel openssl-devel ncurses-devel sqlite-devel readline-devel tk-devel gdbm-devel db4-devel libpcap-devel xz-devel expat-devel

wget  http://python.org/ftp/python/2.7.14/Python-2.7.14.tar.xz

tar xf Python-2.7.14.tar.xz

cd Python-2.7.14

./configure --prefix=/usr/local --enable-unicode=ucs4 --enable-shared LDFLAGS="-Wl,-rpath /usr/local/lib"

# Using make altinstall here so we don't override the system python
make && make altinstall
{% endhighlight %}

If successful, you should get /usr/local/bin/python2.7. The next step would be to install pip:

{% highlight console linenos %}
wget https://bootstrap.pypa.io/get-pip.py

python2.7 get-pip.py
{% endhighlight %}

The above should install pip2.7.

## Virtualenv setup

The next step would be to setup virtualenv so we can use python2.7.14 in its own isolated environment without any conflicts with the system python. Then we can install and build cerbot:

{% highlight console linenos %}
pip2.7 install virtualenv

# create a virtualenv in your home directory called mycertbot
virtualenv mycertbot

# activate the virtualenv
source ~/mycerbot/bin/activate

# should show 2.7.14
python --version

pip install cerbot
{% endhighlight %}

If successful, the certbot binary should be within `~/mycertbot/bin/cerbot`

To test that it works, you can do a __dry-run__ using certbot:

{% highlight console linenos %}
sudo ~/mycertbot/bin/certbot certonly --dry-run --webroot -w /home/web/site/.well-known/acme-challenge -d mydomain.com -d www.mydomain.com
{% endhighlight %}

From above, I'm using the __certonly__ command to generate the certs for each of the domains listed. I'm also using the __webroot__ authenticator.

This means that I only need to supply a directory where __cerbot__ can have access to for storing its acme challenge tokens. In this mode, Certbot does not update or modify the server config files. You would have to do so manually.

Certbot has support for various plugins including nginx and apache. However, I am unable to get it to work properly during the renewal process. I will document the renewal process in the follow up article.

To generate the certificates for the first time, run the command above without the dry-run flag:

{% highlight console linenos %}
sudo ~/mycertbot/bin/certbot certonly --webroot -w /home/web/site/.well-known/acme-challenge -d mydomain.com -d www.mydomain.com
{% endhighlight %}

After a successful run, the certificates should be within `/etc/letsencrypt/live/mydomain.com`.

That's my approach for solving the python dependencies errors when install certbot on centos7.

Hope it helps someone. Happy Hacking!
