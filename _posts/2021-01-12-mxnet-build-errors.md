---
layout:     post
show_meta: true
title:      Building / Compiling MXNet errors
header:     Building / Compiling MXNet errors
date:       2021-01-12 00:00:00
summary:  Common errors encountered with building MxNet
categories: mxnet machine-learning
author: Chee Yeo
---

MXNet is a popular machine learning / deep learning framework. Compared to its peers such as TensorFlow, it is not as popular. 

Indeed, my last search on safari books online only yielded publications with the `mxnet` name mentioned but no concrete examples on how to build or use it.

In this post I aim to show some of the common errors I encountered while building it manually. In future posts, I will demonstrate how I build it from scratch using multi-stage builds.

Below is a list of such errors I encountered.

Please note that its not meant to be a comprehensive / exhaustive list and as usual, different system setup and requirements may / may not result in different errors than mine.

### Compilation Failures

In my earlier attempts at compilation, the process would fail suddenly with errors such as:

{% highlight console linenos %}
...
c++: internal compiler error: Killed (program cc1plus)
Please submit a full bug report,
with preprocessed source if appropriate.
See <file:///usr/share/doc/gcc-7/README.Bugs> for instructions.
make[2]: *** [CMakeFiles/mxnet_static.dir/src/operator/tensor/indexing_op.cc.o] Error 4
make[2]: *** Waiting for unfinished jobs....
...
{% endhighlight %}

This is related to a [gcc 7.5 memory leak issue]{:target="_blank"}

Assuming you have gcc 8 installed, we can use gcc 8 by passing in the following options during compilation...

{% highlight console linenos %}
export CC="gcc-8" && \
export CXX="g++-8"
{% endhighlight %}

Examples show compilation through `ninja` build tool. However, my attempts at using it have been unsuccessful. I find that at least on my local machine, `ninja` tends to consume more CPU and memory resources than it should whereas with `cmake` build tool, I have more control over the number of processes and it also shows the logs in the console than using the former.

### ONNX Issues

If compilation fails with `namespace not found` with onnx option set, then set the appropriate environment variable:

{% highlight console linenos %}
export ONNX_NAMESPACE=onnx
{% endhighlight %}

After compilation but while loading the framework, if it fails with `File already exists: ... onnx-ml.proto`, this is due to the `protobuf` library being built as a shared library and since `onnx-ml.proto` is also symlinked by other libs, it will raise an error.

The only solution to this is to remove all previous installs of protobuf and build it again manually. The below works for me:

{% highlight console linenos %}
git clone --recursive -b 3.5.1.1 https://github.com/google/protobuf.git && \
    cd protobuf && \
    ./autogen.sh && \
    ./configure --disable-shared CXXFLAGS=-fPIC --prefix=/protobufbuild && \
    make -j4 && \
    make install && \
    ldconfig && \
    cd / && \
    cp /protobufbuild /usr/local && \
    rm -rf protobuf
{% endhighlight %}

Note the `--disable-shared` option which builds it as a static library.

For running onnx in python, we need to install the `onnx` pypi package. A compatible version I found to work across all mxnet versions from v1.6.x - v1.8.x is `onnx==1.3.0`

### MKLDNN Issues

MKLDNN is the intel graphics driver for running ML operations on Intel-compatible CPUs. It's a suitable replacement for NVIDIA gpus. More information on [Intel MKLDNN]{:target="_blank"}

The errors below only apply for me when I was trying to compile a python wheel of mxnet. It may not be applicable in your use case.

While compiling the python wheel, I had to enable TVM and it threw an error of:

{% highlight console linenos %}
Traceback (most recent call last):
    File "/mxnet/contrib/tvmop/compile.py", line 20, in <module>
      import tvm
    File "/mxnet/3rdparty/tvm/python/tvm/__init__.py", line 36, in <module>
      from . import target
    File "/mxnet/3rdparty/tvm/python/tvm/target.py", line 70, in <module>
      raise err_msg
    File "/mxnet/3rdparty/tvm/python/tvm/target.py", line 66, in <module>
      from decorator import decorate
  ModuleNotFoundError: No module named 'decorator'
{% endhighlight %}

Ensure that the `decorator==4.4.2` pypi package is present before building the 3rd party plugins.

Another issue I encountered was [not being able to locate / load the TVM config file issue]{:target="_blank"}

After building mxnet, you need to run the following to copy the generated `tvmop.conf` file from the build folder into `/usr/local/lib/<python version>/lib`:

{% highlight console linenos %}
mkdir -p /usr/local/lib/python3.6/lib && \
cp tvmop.conf /usr/local/lib/python3.6/lib/
{% endhighlight %}


### CUDA versions

Another point to note is that for v1.8.x and above, it uses cuda version 10.2 and above. If you are building multiple versions of MXNet in Docker your Dockerfile would need to take into account the different versions required else compilation will not work.

### Conclusion

In conclusion, it takes some effort to get MXNet to compile from source but you will learn a lot about the framework just by doing it - I certainly did.

Happy Hacking.

[gcc 7.5 memory leak issue]: https://github.com/apache/incubator-mxnet/issues/13773

[not being able to locate / load the TVM config file issue]: https://github.com/apache/incubator-mxnet/issues/16704

[Intel MKLDNN]: https://mxnet.apache.org/versions/1.7.0/api/python/docs/tutorials/performance/backend/mkldnn/mkldnn_readme.html