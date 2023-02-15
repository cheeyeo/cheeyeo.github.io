---
layout: post
show_meta: true
title: Compiling OpenCV
header: How to manually compile opencv using nvidia deep learning containers for fun and profit
date: 2023-02-09 00:00:00
summary: How to manually compile opencv using nvidia deep learning containers for fun and profit
categories: docker opencv machine-learning github-actions
author: Chee Yeo
---
[OpenCV DNN GPU example]: https://pyimagesearch.com/2020/02/03/how-to-use-opencvs-dnn-module-with-nvidia-gpus-cuda-and-cudnn/

[nvidia container runtime]: https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html#docker

[OpenCV dockerhub images]: https://hub.docker.com/r/m1l0/opencv

[NVIDIA Deep Learning Containers]: https://docs.nvidia.com/deeplearning/frameworks/user-guide/index.html

In a former project, I attempted to automate the process of building base docker images for machine learning projects. In this post, I will attempt to explain the approach I took for automating the process of building computer vision projects.

OpenCV is a library commonly used in computer vision tasks. You can run `pip install opencv-contrib` to download the latest python packages and this would be adequate for most learning tasks. However, to utilise some of its advanced features such as the **dnn** module which enables one to import and run pre-trained models from other frameworks, you would need to manually compile OpenCV. This would also involve compiling it with the required CUDA / CUDNN libraries.

The overall criteria of the build process becomes:
* The image would need to support both python bindings and C++ libraries.

* The image would need to have the required CUDA/CUDNN Libs installed in order to use `dnn`

* The image would need to be as compact as possible to only include the required libraries due to size constraint.

I decided to split the images into 2 categories: the plain CPU build and the GPU build.

The GPU build process is a **multi-stage build** 

The GPU build is specified in a separate Dockerfile and makes use of the following [NVIDIA Deep Learning Containers]:

* nvidia/cuda:${CUDA}-cudnn8-devel-ubuntu${UBUNTU}
* nvidia/cuda:${CUDA}-cudnn8-runtime-ubuntu${UBUNTU}

According to the documentation, the nvidia images are organized into the following categories:

* **base**
  
  Base image which is built on by other image types

* **devel**
  For development as it contains the necessary compilers and libraries to build applications. Images in this category are big. For example, the `11.8.0-cudnn8-devel-ubuntu22.04` has a size of 8.99 GB uncompressed.

* **runtime**

  For deploying complied applications. This image would only contain the required libraries but without the compilation tools of the devel images, making it smaller in size.

The **cudnn8-devel** images include the CUDNN libraries from which the initial build stage starts. This stage would:

* Install the opencv deps
* Build and install python
* Create a virtualenv to install the python deps and opencv bindings
* Download and compile the opencv source with CUDA enabled.

To build python I utilise a custom script which install the deps and build it from source. Then I declared an env variable for the virtualenv path and install the pip deps:

{% highlight shell %}
...

ENV VIRTUAL_ENV=/opt/venv
ENV PATH="$VIRTUAL_ENV/bin:$PATH"

RUN /usr/local/bin/python -m venv ${VIRTUAL_ENV} && \
  pip --no-cache-dir install --upgrade pip setuptools && \
  pip install --no-cache-dir --upgrade "cmake>=3.13.2" && \
  pip install --no-cache-dir numpy && \
  pip install --no-cache-dir imutils
...

{% endhighlight %}

The virtualenv can be copied during the second stage of the multi-stage build.

To build OpenCV with CUDA support, we need to enable the following CMake flags:

{% highlight shell %}
...
-D WITH_CUDA=ON \
-D WITH_CUDNN=ON \
-D OPENCV_DNN_CUDA=ON \
-D ENABLE_FAST_MATH=1 \
-D CUDA_FAST_MATH=1 \
-D CUDA_ARCH_BIN=6.1 \
...
{% endhighlight %}

The **CUDA_ARCH_BIN** is set to a fixed value as the default includes older compute capability such as 35 which will be deprecated and also slows down the build. My initial understanding is that setting it to a lower value means that it will be compatible with later versions of GPU?

The complete CMake config becomes:
{% highlight shell %}
...

cmake -D CMAKE_BUILD_TYPE=RELEASE \
	-D CMAKE_INSTALL_PREFIX=/installed \
	-D PYTHON_EXECUTABLE=$(which python) \
	-D INSTALL_PYTHON_EXAMPLES=OFF \
	-D OPENCV_GENERATE_PKGCONFIG=ON \
	-D BUILD_opencv_python3=ON \
	-D HAVE_opencv_python3=ON \
	-D OPENCV_CUDA_FORCE_BUILTIN_CMAKE_MODULE=ON \
	-D WITH_CUDA=ON \
	-D WITH_CUDNN=ON \
	-D OPENCV_DNN_CUDA=ON \
	-D ENABLE_FAST_MATH=1 \
	-D CUDA_FAST_MATH=1 \
	-D CUDA_ARCH_BIN=6.1 \
	-D OPENCV_ENABLE_NONFREE=ON \
	-D WTIH_CUBLAS=ON \
	-D WITH_V4L=ON \
	-D BUILD_EXAMPLES=ON \
	-D INSTALL_C_EXAMPLES=OFF \
	-D OPENCV_EXTRA_MODULES_PATH=/opencv_contrib/modules ..
{% endhighlight %}

Once compiled successfully, we symlink the opencv libs into the `site-packages` directory of the python virtualenv:
{% highlight shell linenos %}
ln -s /installed/lib/python${PYVER}/site-packages/cv2/python-${PYVER}/cv2.cpython-${CPYTHON}-x86_64-linux-gnu.so $VIRTUAL_ENV/lib/python${PYVER}/site-packages/cv2.so
{% endhighlight %}

The second stage of the GPU build involves copying the build artifacts from the previous stage into a new image that has the required deps installed. I used the **cudnn8-runtime** image as it has CUDNN and CUDA prebuilt. 

For this stage, I ran the custom scripts to install python and the required OpenCV deps. Then I copy the built OpenCV artifacts and the virtualenv across. Assuming the first stage is called `builder`:

{% highlight shell %}
ENV VIRTUAL_ENV=/opt/venv
ENV PATH="$VIRTUAL_ENV/bin:$PATH"
...

COPY --from=builder /installed /installed

COPY --from=builder /opt/venv /opt/venv
{% endhighlight %}

Note that we still need to declare and append the virtual env path to the system path globally for virtualenv to work across images.

To support C++ compilation, we need to symlink the generated pkg-config file from the OpenCV compilation into the **/usr/share/pkgconfig**. We also need to create an entry in **/etc/ld.so.conf.d/opencv4.conf** so that compiled executables can locate the shared object libraries during load:
{% highlight shell %}
cd /usr/share/pkgconfig && \
ln -s /installed/lib/pkgconfig/opencv4.pc opencv4.pc && \
echo "/installed/lib" >> /etc/ld.so.conf.d/opencv4.conf && \
ldconfig
{% endhighlight %}

The '/installed' directory is where the opencv built artifacts are located and the `opencv4.pc` file is generated when we enabled `-D OPENCV_GENERATE_PKGCONFIG=ON` in the CMake config.

The final image is approx 6.28 GB uncompressed and about 2.78 GB after upload to docker hub.

To run a CUDA image locally using the host GPU, you would need to have the [nvidia container runtime] installed and working first.

I used the following [OpenCV DNN GPU example] to test if the image works locally by mounting the directory of the example code into a running container and running the example:
{% highlight shell %}
docker run -it --rm \
  -v ./opencv-dnn-gpu:/opencv-dnn-gpu \
  --runtime nvidia \
  --gpus all m1l0/opencv:4.5.5-cuda11.8.0-cudnn8-python3.10.9-ubuntu22.04 /bin/bash


# from within running container
cd /opencv-dnn-gpu

python ssd_object_detection.py --prototxt MobileNetSSD_deploy.prototxt \
  --model MobileNetSSD_deploy.caffemodel \
  --input guitar.mp4 --output output2.avi \
  --display 0 \
  --use-gpu 1
{% endhighlight %}

If OpenCV is compiled properly, the above should run and generate the following:

{% highlight shell %}
[INFO] setting preferable backend and target to CUDA...
[INFO] accessing video stream...
[INFO] elasped time: 4.54
[INFO] approx. FPS: 54.38
{% endhighlight %}

Note that we are able to obtain at least 54 FPS for object detection which is impressive.

### Issues during Build

#### Forward compatibility was attempted on non supported HW
{% highlight shell %}
  cv2.error: OpenCV(4.5.5) /opencv/modules/dnn/src/cuda4dnn/csl/memory.hpp:54: error: (-217:Gpu API call) forward compatibility was attempted on non supported HW in function 'ManagedPtr'
{% endhighlight %}

The nvidia container runtime is dependent on the host's nvidia driver version. In this instance my driver is set to `470` but I'm trying to run CUDA 11.8 which requires driver version of `520` and above. I updated the host driver to `525` and the issue was resolved.

#### Could not load library libcudnn_cnn_infer.so.8
{% highlight shell %}
 Could not load library libcudnn_cnn_infer.so.8. Error: libnvrtc.so
{% endhighlight %}

This error occured during the inference stage when calling `network.predict` in the python script.

Running `apt-get -y install cuda-nvrtc-11-8 cuda-nvrtc-dev-11-8` solves the issue

### Remaining Tasks / Improvements 

Additional tasks / improvements could include:

* Updating the Github Action workflow to publish the images to dockerhub

* Automate the image security scan process. Currently this is done locally via `docker scan`

The built images can be found at [OpenCV dockerhub images]{:target="_blank"}.

H4ppy H4ck1ng !!!