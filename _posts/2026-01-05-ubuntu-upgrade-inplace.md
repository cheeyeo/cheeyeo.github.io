---
layout: post
show_meta: true
title: Upgrading Ubuntu
header: Upgrading Ubuntu 
date: 2026-01-05 00:00:00
summary: How to upgrade Ubuntu 20.04 to 22.04 LTS release in place
categories: ubuntu upgrade
author: Chee Yeo
---

In previous attempts to update Ubuntu, I always back up and format the hard drive before reinstalling the new version of Ubuntu, known as a `clean install`. When trying to update Ubuntu from 20.04 to 22.04, I decided to perform an in place update instead using the `update-manager-core` provided by Ubuntu.

The `update-manager-core` is used by the desktop GUI `Update Manager` to manage package sources and release upgrade paths. It also contains the `do-release-upgrade` utility to perform version upgrades.

Firstly, we need to install `update-manager-core`:
{% highlight shell %}
sudo apt update
sudo apt upgrade

sudo apt install update-manager-core
{% endhighlight %}

To start the version upgrade, we invoke the `do-release-upgrade` utility:
{% highlight shell %}
  sudo do-release-upgrade
{% endhighlight %}

In my use case, the above failed with insufficient space in **/boot** which is where all the previous linux kernel versions are kept. To create sufficient space, I need to delete older kernel versions that are no longer in use. As a precaution, I kept at least 2 of the most recent kernel versions:

{% highlight shell %}
sudo dpkg -l | grep linux-image

sudo apt purge <linux kernel name>

sudo dpkg -l | grep linux-headers

sudo apt purge <linux header>

sudo apt autoremove --purge
{% endhighlight %}

Then I restarted the upgrade process again:
{% highlight shell %}
  sudo do-release-upgrade
{% endhighlight %}

Note that you will need to follow the on screen instructions by responding to the CLI. The upgrade tool will automatically walk through the upgrade process by asking the user to respond to certain changes such as upgrading services and removing old system packages.

Once completed, the system will prompt you to restart the system. Press `Y` to complete the process.

To confirm upgrade was successful:
{% highlight shell %}
lsb_release -a # checks version of ubuntu installed

uname -mrs # checks kernel version
{% endhighlight %}
