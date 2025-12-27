---
layout: post
show_meta: true
title: Upgrading Ubuntu
header: Upgrading Ubuntu 
date: 2025-12-31 00:00:00
summary: How to upgrade Ubuntu 20.04 to 22.04 LTS release
categories: ubuntu upgrade
author: Chee Yeo
---


How to update Ubuntu from 20.04 to 22.04


```

sudo apt update
sudo apt upgrade

sudo apt install update-manager-core

sudo do-release-upgrade
```
Above failed with insufficient space in `/boot` which is where all the previous linux kernel versions are kept.

To delete the older versions:
```
sudo dpkg -l | grep linux-image

sudo apt purge <linux kernel name>

sudo dpkg -l | grep linux-headers

sudo apt purge <linux header>

sudo apt autoremove --purge
```

Need to keep at least 2 versions...

To start upgrade:
```
sudo do-release-upgrade
```

Follow on screen instructions. Upgrade tool will automatically walk through the upgrade process by asking user to respond to certain changes such as upgrading services and removing old system packages

Once completed, the system will prompt you to restart the system. Press `Y` to complete the process.

To confirm upgrade was successful:
```
lsb_release -a # checks version of ubuntu installed

uname -mrs # checks kernel version
```



