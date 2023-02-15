---
layout:     post
show_meta: true
title:      Types of Autoencoders Part 2 - Denoising Autoencoder
header:     Types of Autoencoders Part 2 - Denoising Autoencoder
date:       2020-03-31 00:00:00
summary:  Introduction to Denoising Autoencoders
categories: machine-learning deep-learning autoencoders tensorflow
author: Chee Yeo
---

In this post, I aim to introduce the concepts behind Denoising Autoencoders (DAE) and how it differs from the other types of autoencoders we have seen so far.

DAE are autoencoders that receives a corrupted data point and trained to predict original, uncorrupted data point as output.

Structurally, DAE are no different than undercomplete/overcomplete autoencoders. The main difference is in the training inputs. In undercomplete/overcomplete autoencoders, we pass in the original datapoint x and attempt to get the encoder to learn a latent representation of the dataset.

In the case of DAE, the original input data has been corrupted, for instance, by adding random Gaussian noise to an image. The corrupted input, with the original input, are fed as a training pair into the DAE i.e. `(~x, x)`.

Given D as decoder, E as encoder, ~x as corrupted datapoint, x as original datapoint, the DAE is trained to minimize the following loss function:
{% highlight python linenos %}
L(D(E(~x)), x)
{% endhighlight %}

which could be rewritten as the negative log likelihood of the reconstruction distribution of the decoder:
{% highlight python linenos %}
L = -log D(x | h=E(~x))
{% endhighlight %}

By minimizing this loss, it encourages the DAE to learn a vector field that estimates the score of the data distribution of x. 

`D(E(~x))` estimates the center of mass of the `clean points x` that could derive from ~x.

Although DAEs are used for denoising, it also learns a good internal representation of the dataset as a side effect of learning to denoise.

In terms of implementation, the model architecture is still the same as an undercomplete autoencoder. The only difference is that the training data now comprises of pairs of (corrupted, clean) datapoints as input. i.e. `(~x, x)`

In my own research and experimentations, I managed to build a simple denoising autoencoder on the MNIST dataset. The image below shows a generated sample of corrupted/clean data pairs the autoencoder was able to learn from.

##### Denoising autoencoder for MNIST
![MNIST Denoise](/assets/img/autoencoders/mnist_output_denoising.png)

I applied the same autoencoder with parameter changes to the [Kaggle Dirty Documents dataset]{:target="_blank"}. The images on the left are the corrupted images with a noisy background and the images on the right represent the cleaned images. Although the autoencoder was able to remove the noise, more tuning and training is required for the autoencoder to recognise the various font types and sizes for each document.

##### Denoising autoencoder for Kaggle dataset
![Kaggle Denoise](/assets/img/autoencoders/kaggle_denoise_dirty_documents.jpg)

For a working example of building a DAE, please refer to this [keras blog post]{:target="_blank"} or this [denoising autoencoder example]{:target="_blank"} for a more up to date example.

In this post, I aim to introduce what a Denoising Autoencoder is and how it differs from other autoencoders.

Happy Hacking.

[Kaggle Dirty Documents dataset]: https://www.kaggle.com/c/denoising-dirty-documents

[keras blog post]: https://blog.keras.io/building-autoencoders-in-keras.html

[denoising autoencoder example]: https://www.pyimagesearch.com/2020/02/24/denoising-autoencoders-with-keras-tensorflow-and-deep-learning/



