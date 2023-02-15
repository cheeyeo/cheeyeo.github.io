---
layout:     post
show_meta: true
title:      Tensorflow 2.0 - Dataset
header:     Tensorflow 2.0 - Dataset
date:       2020-04-08 00:00:00
summary:  Introduction to using tf.data.Dataset
categories: machine-learning tensorflow dataset
author: Chee Yeo
---

This is a series of posts exploring some of the new features in tensorflow 2.0, which I am currently using in my own projects. These posts are introductory guides and do not cover more advanced uses.

Tensorflow 2.0 introduced the concept of a `Dataset`. This high level API allows you to load different data formats such as images, numpy arrays and panda dataframes.

Previously, in Keras, when we want to load a training dataset that is too big to fit into memory, we create a custom generator that iterates over the dataset in batches which are fed into the model during training using method calls such as `fit_generator`.

The issue with the above approach is that it can be error-prone to setup. For instance, changes to the dataset structure means changes to the generator or there could be issues in the generator code implementation.

A `Dataset` is a high-level construct in TF 2.0 which represent a collection of data or documents. It supports batching, caching and pre-fetching of data in the background. The dataset is not loaded into memory but streamed into the model when its iterated through.

Using a `Dataset` generally follows the guidelines:

* Create a dataset from input data

* Apply transformations to preprocess the data

* Iterate over dataset and process its elements i.e. training loop

Let's go through each of the above stages in the pipeline.

### Creating a dataset

The easiest method to create a dataset is to use the `from_tensor_slices` method:

{% highlight python linenos %}
dataset = tf.data.Dataset.from_tensor_slices([1,2,3])
for ele in dataset:
  print(ele) # returns tf.Tensor
{% endhighlight %}

If we try to print each element of a dataset, we get a `Tensor` object back. In order to inspect the contents, we can call the `as_numpy_iterator` method to convert the tensors into numpy arrays, which returns an iterable:

{% highlight python linenos %}
for num in dataset.as_numpy_iterator():
  print(num)
{% endhighlight %}

To create dataset from a directory list of files, we can use the `list_files` method which accepts a file/glob matching pattern. For example, if we had a directory of `"/mydir/"`, consisting of python files such as `"/mydir/a.py", "/mydir/b.py"`, it would produce the following:

{% highlight python linenos %}
dataset = tf.data.Dataset.list_files("/mydir/*.py")
files_list = list(dataset.as_numpy_iterator())
print(files_list) # => returns ["/mydir/a.py", "/mydir/b.py"]
{% endhighlight %}

The issue with the above approach is that globbing occurs for every filename encountered in the path, so its more efficient to produce the list of file names first and construct the dataset using `from_tensor_slices`

There are other methods such as `from_generator` and `from_tensors` which are outside the scope of this article. We will be using `from_tensor_slices` in a working example below.

### Apply transformations to dataset

Now that we have a dataset of elements, the next step would be to preprocess it. We can call the `map` method and pass a function to process each element.

For instance, we may want to resize each image and perform mean normalization as part of preprocessing.

{% highlight python linenos %}
# list_of_files is a collection of file paths...
dataset = tf.data.Dataset.from_tensor_slices(list_of_files)

train_ds = dataset.map(process_img)

def process_img(file_path):
  # read and process the image
  img = tf.io.read_file(file_path)
  img = tf.image.decode_jpeg(img, channels=3)
  # mean normalization
  img = tf.image.convert_image_dtype(img, tf.float32)
  img /= 255.0
  # resize the image
  img = tf.image.resize(img, (64, 64))
  return img
{% endhighlight %}

After calling `process_img` in the above, `train_ds` will now contain a dataset of preprocessed images.

Since `map` returns a dataset, we can chain multiple calls together, clarifying the sequence of operations:

{% highlight python linenos %}
def func1(x):
  return x * 2

def func2(x):
  return x ** 2

ds = tf.data.Dataset.from_tensor_slices([1,2,3])

new_ds = ds.map(func1).map(func2)

print(list(new_ds.as_numpy_iterator())) # => [4, 16, 36]
{% endhighlight %}

### Iteration over dataset

We need to set certain parameters on the dataset object before we can pass it into a model for training. This would include setting the batch size, caching, pre-fetching options.

Using the image classification example above, we can do the following:

{% highlight python linenos %}
dataset = tf.data.Dataset.from_tensor_slices(list_of_files)
train_ds = dataset.map(process_img)
train_ds = train_ds.shuffle(buffer_size=1024).batch(64)

model.fit(train_ds, epochs=3)
{% endhighlight %}

The `shuffle` function randomly shuffles the elements in the dataset. The `batch` function sets the batch size for each training epoch. Note that by using `batch` we don't have to set the batch size argument in the `fit` function.

One can also chain further functions such as `cache` to cache the data in memory or on the filesystem by setting the filename argument in the function. This is extremely useful when training large datasets. 

Note that, the first iteration of the training loop will create the cache, after which, subsequent runs will use the same cached data in the same sequence. To randomize the data between iterations, call `shuffle` after `cache`

For example:
{% highlight python linenos %}
train_ds = train_ds.cache("cache/mycache").shuffle(buffer_size=1024).batch(64)
{% endhighlight %}

When the training loop is restarted, the cache directory needs to be cleared else it will raise an exception.

For most training scenarios, passing the dataset into `model.fit` will be sufficient. However, if you do have a custom/manual training process where you are iterating the dataset across multiple epochs, you need to call `repeat` before `batch` to iterate over the dataset.

{% highlight python linenos %}
train_ds = train_ds.repeat().batch(64)

for ele in train_ds.as_numpy_iterator():
  print(ele)
{% endhighlight %}

To access the next batch of data, you can create an iterator from the dataset by calling `as_numpy_iterator` or wrapping the dataset object in `iter()` and call `next` to retrieve the next batch of data.

For a working implementation, please refer to the following example on [applying tf.data.Dataset on MNIST]{:target="_blank"}. The [tf.data.Dataset API]{:target="_blank"} has more details on the various functions and examples.

Happy Hacking!


[applying tf.data.Dataset on MNIST]: https://www.tensorflow.org/guide/keras/train_and_evaluate#training_evaluation_from_tfdata_datasets

[tf.data.Dataset API]: https://www.tensorflow.org/api_docs/python/tf/data/Dataset