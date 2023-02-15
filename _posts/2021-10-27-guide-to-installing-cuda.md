---
layout: post
show_meta: true
title: Guide to installing CUDA on Ubuntu 18.04 LTS
header: Guide to installing CUDA on Ubuntu 18.04 LTS
date: 2021-10-27 00:00:00
summary: Pain-free guide to installing CUDA 11 
categories: cuda machine-learning mlops
author: Chee Yeo
---

[blog post on pyimagesearch on installing Tensorflow 2.0]: https://www.pyimagesearch.com/2019/12/09/how-to-install-tensorflow-2-0-on-ubuntu/

[prebuilt Tensorflow docker image]: https://hub.docker.com/r/tensorflow/tensorflow/tags


This guide attempts to highlight a process of installing CUDA 11 on UBUNTU 18.04 LTS.

This post is inspired by the following [blog post on pyimagesearch on installing Tensorflow 2.0]{:target="_blank"}.

Its not by any means a comprehensive guide as hardware differs but it aims to hopefully get you up and running asap.

The steps documented below applies for both new installs or to update an existing CUDA install.


### Hardware and operating system tested

Ubuntu 18.04 LTS with GeForce GTX 1060


### Pre-install step

Update and install system dependencies:
{% highlight console %}

sudo apt update && sudo apt upgrade

sudo apt install build-essential \
cmake \
unzip \
pkg-config \
gcc-7 \
g++-7 \
libopenblas-dev \
libatlas-base-dev \
liblapack-dev \
gfortran \
python3-dev \
python3-tk \
python-imaging-tk
{% endhighlight %}


### Update / Install nvidia device driver

Add PPA for nvidia device drivers and install **nvidia-driver-470**

CUDA 11.3 only works for versions of device drivers >= 465. 

Note: for Ubuntu, the 465 driver is linked to the 470 driver so there is no dedicated 465 version

Install device driver:
{% highlight console %}
sudo add-apt-repository ppa:graphics-drivers/ppa

sudo apt-get install nvidia-driver-470
{% endhighlight %}


After install/update, reboot the system.

Check that the device driver is working by running nvidia-smi

{% highlight console %}
nvidia-smi
{% endhighlight %}

If it works, you should see the device driver version and the GPU hardware in the display.

As an additional sanity check, you can also bring up **NVIDIA Xserver settings** and check that it has picked up the right device driver version and GPU.

Note that the nvidia-smi utility is installed through the drivers and is independent of CUDA.


### Installing CUDA

For the purposes of this guide we are installing CUDA 11.3 in order to install and run tensorflow 2.6.0. This is to overcome the issue of the missing **libcudart.11.0** library.

For CUDA 11.3, you need the device driver to be at least >= 465, hence we installed 470 of the driver above.

Easiest way to install CUDA is to download and run the installer.

{% highlight console %}
wget https://developer.download.nvidia.com/compute/cuda/11.3.0/local_installers/cuda_11.3.0_465.19.01_linux.run

chmod +x cuda_11.3.0_465.19.01_linux.run

sudo ./cuda_11.3.0_465.19.01_linux.run --override

{% endhighlight %}

This will bring up an install screen. **Uncheck** the 465 driver option. This is **IMPORTANT** else it will corrupt the device driver since we have already installed it in step 1.

Keep the remaining options as it is.

After installation, it will copy the cuda libs to `/usr/local/cuda-11.3` and makes a symlink to `/usr/local/cuda`

To check that cuda is installed, run **nvcc** compiler:
{% highlight console %}
nvcc --version
{% endhighlight %}

Note that the CUDA version reported in nvidia-smi will not match the current installed version.

Update **~/.bashrc** by setting the LD_PATH and PATH variables for cuda:

{% highlight console %}
export PATH=/usr/local/cuda/bin:$PATH
export LD_LIBRARY_PATH=/usr/local/cuda/lib64
{% endhighlight %}


### Install CUDNN

The CUDNN library is required by tensorflow.

The approach I took was to install using the deb file from the nvidia cuda repo. The version of libcudnn after the **+** symbol has to match with the installed cuda version.

For example, if we have cuda 11.3 then we need to install the deb files with **..+cuda11.3..** in the suffix.

{% highlight console %}
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu1804/x86_64/libcudnn8_8.2.1.32-1+cuda11.3_amd64.deb

wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu1804/x86_64/libcudnn8-dev_8.2.1.32-1+cuda11.3_amd64.deb

sudo dpkg -i libcudnn8_8.2.1.32-1+cuda11.3_amd64.deb

sudo dpkg -i libcudnn8-dev_8.2.1.32-1+cuda11.3_amd64.deb

sudo ldconfig
{% endhighlight %}


**NOTE:** You need to ensure that you don't have existing versions of CUDNN before installing a newer version. TF will pick up the older version and will throw a `mismatch CUDNN version` error during invocation.


### Install TensorRT libs

To run TensorRT, we need to install the libnvinfer libraries. These would require cuda-nvrtc libraries as dependencies else the install would fail.

Again, ensure that the cuda versions in the filenames match the actual installed cuda version.

{% highlight console %}
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu1804/x86_64/cuda-nvrtc-11-3_11.3.58-1_amd64.deb

wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu1804/x86_64/cuda-nvrtc-dev-11-3_11.3.58-1_amd64.deb

wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu1804/x86_64/libnvinfer8_8.0.3-1+cuda11.3_amd64.deb

wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu1804/x86_64/libnvinfer-dev_8_8.0.3-1+cuda11.3_amd64.deb

wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu1804/x86_64/libnvinfer-plugin8_8.0.3-1+cuda11.3_amd64.deb

wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu1804/x86_64/libnvinfer-plugin-dev_8_8.0.3-1+cuda11.3_amd64.deb

sudo dpkg -i cuda-nvrtc-11-3_11.3.58-1_amd64.deb \
cuda-nvrtc-dev-11-3_11.3.58-1_amd64.deb \
libnvinfer8_8.0.3-1+cuda11.3_amd64.deb \
libnvinfer-dev_8_8.0.3-1+cuda11.3_amd64.deb \
libnvinfer-plugin8_8.0.3-1+cuda11.3_amd64.deb \
libnvinfer-plugin-dev_8_8.0.3-1+cuda11.3_amd64.deb

sudo ldconfig
{% endhighlight %}


### Install Tensorflow

I tend to create a venv to test any new install of tensorflow as it has multiple dependencies which may or may not conflict with existing packages.

Firstly, we need to export the LD_PATH to include CUPTI from CUDA. Then we create a venv and install tensorflow:

{% highlight python lineanchors %}
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/cuda/extras/CUPTI/lib64

python3 -m venv /path/to/venv

source /path/to/venv activate

pip install tensorflow==2.6.0
{% endhighlight %}


### Test tensorflow install

While still in activated venv, run following test script:

{% highlight python lineanchors %}
import tensorflow as tf

if __name__ == "__main__":
    print("[INFO] Checking TF Gpu installed ...")
    print(tf.config.list_physical_devices("GPU"))
    print("TF VERSION: {}".format(tf.__version__))

    tf.debugging.set_log_device_placement(True)

    a = tf.constant([[1.0, 2.0, 3.0], [4.0, 5.0, 6.0]])
    b = tf.constant([[1.0, 2.0], [3.0, 4.0], [5.0, 6.0]])
    c = tf.matmul(a, b)

    print(c)
{% endhighlight %}

If all goes well, should see output similar to this:
{% highlight python lineanchors %}
....

[PhysicalDevice(name='/physical_device:GPU:0', device_type='GPU')]

TF VERSION: 2.6.0

....

2021-10-27 15:01:45.114704: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1510] Created device /job:localhost/replica:0/task:0/device:GPU:0 with 5018 MB memory:  -> device: 0, name: NVIDIA GeForce GTX 1060 with Max-Q Design, pci bus id: 0000:01:00.0, compute capability: 6.1
2021-10-27 15:01:45.172025: I tensorflow/core/common_runtime/eager/execute.cc:1161] Executing op _EagerConst in device /job:localhost/replica:0/task:0/device:GPU:0
2021-10-27 15:01:45.172447: I tensorflow/core/common_runtime/eager/execute.cc:1161] Executing op _EagerConst in device /job:localhost/replica:0/task:0/device:GPU:0
2021-10-27 15:01:45.173005: I tensorflow/core/common_runtime/eager/execute.cc:1161] Executing op MatMul in device /job:localhost/replica:0/task:0/device:GPU:0
tf.Tensor(
[[22. 28.]
 [49. 64.]], shape=(2, 2), dtype=float32)

{% endhighlight %}

Important lines are **device /job:localhost/replica:0/task:0/device:GPU:0** which indicates that TF is able to locate and access the GPU device.

### Pre-built docker images

An alternative is to run tensorflow locally using one of the [prebuilt Tensorflow docker image]{:target="_blank"} and bind-mount a local directory into the running container:

{% highlight console %}
docker pull tensorflow/tensorflow:2.6.0

docker run --gpus all -it --rm -v <source path>:<target container path> --entrypoint /bin/bash tensorflow/tensorflow-gpu:2.6.0

cd <target container path>
{% endhighlight %}