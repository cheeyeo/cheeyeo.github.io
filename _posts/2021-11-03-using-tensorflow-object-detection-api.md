---
layout: post
show_meta: true
title: Guide to training your own object detector using the TFOD API V2
header: Guide to training your own object detector using the TFOD API V2
date: 2021-11-03 00:00:00
summary: Pain-free guide to using the TFOD API
categories: machine-learning deep-learning computer-vision tensorflow
author: Chee Yeo
---

[LISA Traffic signs dataset]: http://cvrr.ucsd.edu/LISA/lisa-traffic-sign-dataset.html

[TensorFlow Object Detection API]: https://github.com/tensorflow/models/tree/master/research/object_detection

[TF2 Model Zoo]: https://github.com/tensorflow/models/blob/master/research/object_detection/g3doc/tf2_detection_zoo.md 

[TFOD setup using TF 2]: https://github.com/tensorflow/models/blob/master/research/object_detection/g3doc/tf2.md

[Faster R-CNN Resnet101 V1 model]: http://download.tensorflow.org/models/object_detection/tf2/20200711/faster_rcnn_resnet101_v1_800x1333_coco17_gpu-8.tar.gz

[example TFOD project]: https://github.com/cheeyeo/tfod_rcnn_example

In my recent studies on computer vision, I come across the Faster-RCNN network, which is widely used in real-time object detection. 

The purpose of this post is to describe how to get up and running with the TFOD framework. I defer a detailed discussion of the Faster-RCNN architecture and how to evaluate and export the trained model in a follow-up post.

### Hardware and OS tested 

Tested on Ubuntu 18.04 LTS with a single GeForce GTX 1060 GPU, 16GB RAM.

### Setup

An [example TFOD project] is created with this blog post to highlight the process.

The [LISA Traffic signs dataset] is used for training and evaluation. 

The dataset consists of 47 different USA traffic sign types. There are 7855 individual annotations. The training images were taken from a dashcam footage with varying levels of quality and resolution.

To limit the amount of resources required for training, we will only be using 3 traffic sign types for training: **stop sign, pedestrian crossing, signal ahead signs**

The dataset will need to be preprocessed into a specific format, namely the TF record format, before it can be used by the TFOD API.

Also the class labels will need to be processed into a specific format before it can be used.

### Data Preprocessing

As mentioned previously, the input data needs to be converted into **tf.train.Example** record with details of each input converted into a **tf.train.Features** object.


The dataset has 47 categories. We are only using three of it and defined it in the project config file as a python dict:
{% highlight python %}
{"pedestrianCrossing": 1, "signalAhead": 2, "stop": 3}
{% endhighlight %}

Next, we need to convert the dict above into the required format:
{% highlight python %}
item {
	id: 1
	name: 'pedestrianCrossing'
}
item {
	id: 2
	name: 'signalAhead'
}
item {
	id: 3
	name: 'stop'
}
{% endhighlight %}

The above is implemented in the **build_lisa_records.py** script in the [example TFOD project].

Each image detail is stored as a CSV row in the **allAnnotations.csv** file in the following format:

* Filename
* Annotation tag ( class label )
* Upper left corner X ( start X )
* Upper left corner Y ( start Y )
* Lower right corner X ( end X )
* Lower right corner Y ( end Y )
* Occluded,On another road ( not used )
* Origin file ( not used )
* Origin frame number ( not used )
* Origin track ( not used )
* Origin track frame number ( not used )

We parse the above CSV file, ignoring the headers and only use the first 6 fields to obtain the filename, label, and bounding box coordinates. The parsed data is stored into a python dict, keyed by the image filename. Each value is a tuple of the form `((label, (startX, startY, endX, endY)))`.

The dictionary keys are passed through to `train_test_split` in scikit-learn to split the dataset into train/test sets. We use a split of 0.75 for training and 0.25 for testing.

For each of the data split, we need to convert each entry into a `tf.train.Example` record with the following required fields for its features, which is a `tf.train.Features` object:

* "image/height"
* "image/width"
* "image/filename"
* "image/source_id" ( filename )
* "image/encoded" ( actual image encoded into bytes )
* "image/format" ( image file type )
* "image/object/bbox/xmin" ( start X coord of bounding box ground truth )
* "image/object/bbox/xmax" ( end X coord of bounding box ground truth )
* "image/object/bbox/ymin" ( start Y coord of bounding box ground truth )
* "image/object/bbox/ymax" ( end Y coord of bounding box ground truth )
* "image/object/class/text" ( string class label )
* "image/object/class/label" ( integer class label )
* "image/object/difficult"

Note that for "image/encoded", we read each image using `tf.io.gfile.GFile` and encode it into bytes.

Note that each of the bounding box coordinate is also normalized to the range [0, 1] by dividing it by its width and height values.

Note that for "image/object/class/label", we refer to the python dict for the classes defined in the config file to obtain an integer representation of the string label.

A custom class `TFAnnotation` is created to generate the above features which are then used to create an individual example object:

{% highlight python %}
writer = tf.io.TFRecordWriter(output_path)
...

features = tf.train.Features(feature=tfannot.build())
example = tf.train.Example(features=features)

writer.write(example.SerializeToString())
{% endhighlight %}

The above is implemented in the `build_lisa_records.py` script.

After preprocessing, we should obtain the following training files:
{% highlight console %}
lisa/records/
├── classes.pbtxt
├── testing.record
└── training.record
{% endhighlight %}

### Setup TFOD

To setup TFOD we need to:

* Clone the models repository
* Generate the protobuf files
* Copy the setup.py file provided and install the dependencies

The above can be summarized as follows:
{% highlight console %}
cd /tfod_example

git clone https://github.com/tensorflow/models.git

cd models/research/

protoc object_detection/protos/*.proto --python_out=.

cp object_detection/packages/tf2/setup.py .

python -m pip install --use-feature=2020-resolver .

cd ../../

setup.sh

python object_detection/builders/model_builder_tf2_test.py
{% endhighlight %}

Note that the `setup.sh` script is needed to add the absolute path of the `models/research` and `models/research/slim` directories to PYTHONPATH in order for the training script to locate the imports.

An example setup.sh script looks as follows:
{% highlight console %}
#!/bin/sh

export PYTHONPATH=$PYTHONPATH:'/tfod_example/models/research':'/tfod_example/models/research/slim'
{% endhighlight %}

Once the provided test script runs successfully, you will have setup TFOD.

The [example TFOD project] has a setup.sh script provided that takes as arguments the required paths and set them up as environment variables.


### Setup pre-trained model

I decided to use the pre-trained [Faster R-CNN Resnet101 V1 model] for this example.

Faster-RCNN describes an architecture whereby a pre-trained base model is used for transfer learning. In this case, we are using ResNet but you can swap it out for other network types such as MobileNet for example.

Download the [Faster R-CNN Resnet101 V1 model] and untar it in a specific directory:
{% highlight console %}
mkdir -p lisa/experiments/training

tar -zxvf faster_rcnn_resnet101_v1_800x1333_coco17_gpu-8.tar.gz
{% endhighlight %}

Note that each model archive has an entry of `gpu` or `tpu` in its filename. You need to select the model based on your own hardware. For instance, for training using Colab, the tpu version should be used. If you have a local gpu to train against, use the gpu version.

The model dir will consist of the following structure:
{% highlight console %}
lisa/experiments/training/faster_rcnn_resnet101_v1_800x1333_coco17_gpu-8/
├── checkpoint
│   ├── checkpoint
│   ├── ckpt-0.data-00000-of-00001
│   └── ckpt-0.index
├── saved_model
│   ├── variables
│   │   ├── variables.data-00000-of-00001
│   │   └── variables.index
│   └── saved_model.pb
└── pipeline.config
{% endhighlight %}

This directory contains the pretrained model's weights and a sample config file. The path to the model's weights is referenced in the training config file below.

We make a copy of the `pipeline.config` file and use it as a starting point for this project.

### Defining the training config

After making a copy of the `pipeline.config` file above, we need to update it with our custom paths as follows. For this example we rename the sample config file as `faster_rcnn_lisa.config`

For the model config, we set `num_classes` to 3 specific for this example.

{% highlight python %}
model {
  faster_rcnn {
    num_classes: 3
    image_resizer {
      keep_aspect_ratio_resizer {
        min_dimension: 600
        max_dimension: 1024
      }
    }
  }
  ....
}
{% endhighlight %}

For the `train_config` block we update the `batch_size`, `num_steps`, and checkpoints parameters:
{% highlight python %}
train_config: {
  batch_size: 1
  num_steps: 50000
  ...

  fine_tune_checkpoint_version: V2
  fine_tune_checkpoint: "/tfod_example/lisa/experiments/training/faster_rcnn_resnet101_v1_800x1333_coco17_gpu-8/checkpoint/ckpt-0"
  from_detection_checkpoint: true
  fine_tune_checkpoint_type: "detection"

  ...
}
{% endhighlight %}

We set the num of training steps to 50000. The original value is 200000 but you would need a minimum of at least 20000 training steps for a baseline model. The batch size is set to 1 for my setup but can be increased to match your existing compute resources. The important configuration are the checkpoint values. Note that the **fine_tune_checkpoint** path must point to the checkpoint file in the model download from before. The filename should be just the prefix i.e. `ckpt-0`. Also set the finetune checkpoint type to "detection".

The next config blocks to change would be for the training and test datasets.

For the train_input_reader block:
{% highlight python %}
train_input_reader: {
  label_map_path: "/tfod_example/lisa/records/classes.pbtxt"
  tf_record_input_reader {
    input_path: "/tfod_example/lisa/records/training.record"
  }
}
{% endhighlight %}

Note that these files are generated during the data preprocessing phase described above. The **label_map_path** refers to the mapping of string class names to its integer value. The **tf_record_input_reader** is the TFRecord training file.

Next we update the eval_input_reader block:
{% highlight python %}
eval_input_reader: {
  label_map_path: "/tfod_example/lisa/records/classes.pbtxt"
  shuffle: false
  num_epochs: 1
  tf_record_input_reader {
    input_path: "/tfod_example/lisa/records/testing.record"
  }
}
{% endhighlight %}

The **tf_record_input_reader** refers to the TFRecord for the test set.

For the eval_config block:
{% highlight python %}
eval_config: {
  metrics_set: "coco_detection_metrics"
  num_examples: 955
}
{% endhighlight %}

We set the num_examples to match the total number of examples in the test set which is provided when running the preprocessing script.

Note that all the paths must be absolute paths.

### Training process

The final step is to run the provided training script from within the TFOD models directory cloned earlier.

{% highlight python %}
source setup.sh

python models/research/object_detection/model_main_tf2.py \
	--pipeline_config_path="/tfod_example/lisa/experiments/training/faster_rcnn_lisa.config" \
	--model_dir="/tfod_example/lisa/experiments/training" \
	--num_train_steps=50000 \
	--sample_1_of_n_eval_examples=1 \
	--alsologtostderr
{% endhighlight %}

You need to setup the **PYTHONPATH** for the TFOD imports before runnning the training script by running `source setup.sh`

Note that we are using the `model_main_tf2.py` script as this example is for TF 2.

If training starts successfully, you should see the following output:
{% highlight console %}

...

INFO:tensorflow:Step 2100 per-step time 0.733s
I1103 12:46:44.766502 140491317245760 model_lib_v2.py:700] Step 2100 per-step time 0.733s
INFO:tensorflow:{'Loss/BoxClassifierLoss/classification_loss': 0.01688414,
 'Loss/BoxClassifierLoss/localization_loss': 0.05560676,
 'Loss/RPNLoss/localization_loss': 0.013312755,
 'Loss/RPNLoss/objectness_loss': 0.008382628,
 'Loss/regularization_loss': 0.0,
 'Loss/total_loss': 0.09418628,
 'learning_rate': 0.0042}
I1103 12:46:44.766725 140491317245760 model_lib_v2.py:701] {'Loss/BoxClassifierLoss/classification_loss': 0.01688414,
 'Loss/BoxClassifierLoss/localization_loss': 0.05560676,
 'Loss/RPNLoss/localization_loss': 0.013312755,
 'Loss/RPNLoss/objectness_loss': 0.008382628,
 'Loss/regularization_loss': 0.0,
 'Loss/total_loss': 0.09418628,
 'learning_rate': 0.0042}

....
{% endhighlight %}

Since there is an existing checkpoint, the trainer has picked up on it from a previous run.

To view the training progress you can use `tensorboard` to point to the training subdirectory as follows:
```
tensorboard --logdir /tfod_example/lisa/experiments/training
```

Note that it may take some time before the mAP metrics show up in the dashboard.

In conclusion, the TFOD is a robust framework to learn for prototyping object detection projects. Due to its complexity, it will take a while to get used to its intricacies but the effort is worth it in my opinion.

For more information, consult the official [TensorFlow Object Detection API] github project page and the official documentation on [TFOD setup using TF 2]. Pre-trained models can be found on [TF2 Model Zoo].

Happy Hacking !