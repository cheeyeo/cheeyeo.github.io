---
layout: post
show_meta: true
title: Fixing thunderbolt3 on Ubuntu 18.04 LTS for eGPU
header: Fixing thunderbolt3 on Ubuntu 18.04 LTS for eGPU
date: 2021-11-23 00:00:00
summary: Pain-free guide to fixing the issue of thunderbolt3 in Ubuntu 18.04
categories: machine-learning thunderbolt3 egpu ubuntu 18.04
author: Chee Yeo
---

[eGPU for Machine Learning]: https://www.cheeyeo.xyz/egpu/ubuntu/18.04/machine-learning/2020/03/13/multi-egpu-ubuntu/

[bolt source]: https://gitlab.freedesktop.org/bolt/bolt/-/releases

On a previous post on using [eGPU for Machine Learning], I described a process of setting up an external GPU for local distributed training of ML models.

It relies on using the **bolt** package provided upstream which is fixed at **0.5.0** for Ubuntu 18.04 LTS. 

After a kernel update to version **4.15.0-163-generic**, the thunderbolt controller was put into a forced shutdown everytime the system starts, resulting in the thunderbolt3 controller not being able to recognise the eGPU.

After some digging, I discovered that the thunderbolt controller is running as a background service under systemctl. This means that we can stop this service and replace it with another one which calls an updated version of bolt.

I removed the system version and resinstalled a later version of boltd to test if this would fix the issue of the forced shutdown detailed above.

The process I took was as follows:

* Download the [bolt source]. I picked version 0.7.0.

* Setup a venv through python as the build stage requires meson, a python package, for compilation.

{% highlight console linenos %}
# make sure in virtual env and install meson and ninja 
python3 -m venv boltvenv
source boltvenv/bin/activate
pip install meson
pip install ninja

sudo apt-get install libpolkit-gobject-1-dev
{% endhighlight %}

* Compile bolt from source as follows:
{% highlight console linenos %}
curl -L https://gitlab.freedesktop.org/bolt/bolt/-/archive/0.7/bolt-0.7.tar.gz -o bolt.tar.gz

tar -zxvf bolt.tar.gz

cd bolt-0.7.0

meson build \
      --sysconfdir=/etc \
      --localstatedir=/var \
      --sharedstatedir=/var/lib
{% endhighlight %}

Note that I left the build parameters as the defaults as documented on the project website.

* Build the bolt package:
{% highlight console linenos %}
ninja -C build

ninja -C build test

sudo ninja -C build install

sudo systemctl daemon-reload
{% endhighlight %}

Restart the system.

If the above goes well, you should have the bolt service running after running **sudo systemctl status bolt**

![Output of systemctl status for bolt](/assets/img/egpu/bolt_systemctl.png)

Running **boltctl --version** should also report version 0.7.0

Next check that the current device (laptop) is registered successfully as a domain under bolt via :
{% highlight console linenos %}
boltctl domains
{% endhighlight %}

![Output of boltctl domains](/assets/img/egpu/bolt_domains.png)


Connect the eGPU, reboot and check that its connected and registered with bolt:
{% highlight console linenos %}
boltctl list
{% endhighlight %}

![Output of boltctl list](/assets/img/egpu/bolt_list.png)

![GNOME Panel view of thunderbolt](/assets/img/egpu/gnome_panel.png)


Note that the screenshots above show that the eGPU has been purposefully disconnected to ensure that the device has been successfully registered on system reboot. Using version 0.5.0 of bolt package, it was blank.

The screenshots also shows that it has managed to register the eGPU on reboot. The GNOME control panel has also displayed the registered eGPU which shows that the bolt service is running properly.

Note that you will need to plugin the eGPU and restart before **nvidia-smi** can recognise the additional GPU.


### Further Notes

If the eGPU is still not recognised you can check that the physical thunderbolt controller is actually enabled under the BIOS.

I ran the following command from the terminal since I can't get into BIOS due to GRUB:

{% highlight console linenos %}
sudo systemctl reboot --firmware-setup
{% endhighlight %}

Check for the presence of thunderbolt3 hardware controller and ensure its enabled.

As an additional measure, I also enabled the option to detect thunderbolt devices with PCIe cards installed to be initiated on startup, if its available as an option in your BIOS settings.


### Summary

The thunderbolt3 controller in Ubuntu runs as a background service of **boltd** which can be upgraded manually to a later version to address issues of force shutdowns. Note that I only tested version 0.7.0. since its the only version I can get to work with the GNOME control panel.


Happy hacking !!!