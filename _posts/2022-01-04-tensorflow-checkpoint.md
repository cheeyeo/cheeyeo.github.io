---
layout: post
show_meta: true
title: Saving Model checkpoint in Tensorflow 2.0 using tf.train.Checkpoint
header: Saving Model checkpoint in Tensorflow 2.0 using tf.train.Checkpoint
date: 2022-01-04 00:00:00
summary: Saving GAN model checkpoints during training in TF 2.0 using tf.train.Checkpoint
categories: machine-learning tensorflow tf2.0 gan
author: Chee Yeo
---

[TF 2.0 Guide on training GAN]: https://www.tensorflow.org/tutorials/generative/dcgan

[tf.train.Checkpoint API]: https://www.tensorflow.org/api_docs/python/tf/train/Checkpoint

Whilst exploring how to build and train a GAN model in tensorflow I came upon an interesting issue on how to save the model's checkpoints during training.

One can usually save a model's checkpoint either using the built-in **ModelCheckpoint** callback or by using a custom callback class subclassing from **Callback**. The issue is with a GAN, we have two models being trained concurrently - the critic / discriminator and the generator.

Normally, we tend to wrap both models inside another class object or we can create a custom model class subclassed from **tf.keras.models.Model** and override the **train_step** function.

In both cases, we can't define a callback on the resultant model using **fit** as it is only a logical wrapper around the critic/generator models.  

If we attempt to define the checkpoint callback on the logical wrapper model we will get the following error:

{% highlight python linenos %}
...

ValueError: Model <models.dcgan.DCGAN object at 0x7f84463f9d10> cannot be saved because the input shapes have not been set. Usually, input shapes are automatically determined from calling `.fit()` or `.predict()`. To manually set the shapes, call `model.build(input_shape)`.

{% endhighlight %}

As per the [TF 2.0 Guide on training GAN], it uses an object of **tf.train.Checkpoint** to save the checkpoints of the optimizers, generator and critic models during training.

Using the above approach, we can create a custom callback that would allow us to save the current checkpoint per epoch:

{% highlight python linenos %}
class EpochCheckpoint(Callback):
    def __init__(self, output_dir, every=5, start_at=0, ckpt_obj=None):
        super(EpochCheckpoint, self).__init__()

        self.checkpoint_dir = output_dir
        self.every = every
        self.int_epoch = start_at
        self.checkpoint = ckpt_obj

    def on_epoch_end(self, epoch, logs=None):
        if (self.int_epoch + 1) % self.every == 0:
            checkpoint_prefix = os.path.join(self.checkpoint_dir, "ckpt")
            self.checkpoint.save(file_prefix=checkpoint_prefix)

        self.int_epoch += 1
{% endhighlight %}


Firstly, we subclass **Callback** class and initialize some instance variables:

* **checkpoint_dir** where the checkpoint is to be saved

* **every**, frequency at which we save per epoch

* **int_epoch**, when to start saving the checkpoint, Defaults to epoch 0

* **checkpoint**, the **tf.train.Checkpoint** object which gets passed from the training script containing the optimizers and models to save

We override the **on_epoch_end** function to save the checkpoints at the end of each epoch. Within the function call, we initialize the prefix to save the checkpoint and calls the **save** method of the checkpoint object. Then we increment the internal counter to track the current epoch number.

Within the main training script, we need to initialize the above callback and define the objects we want the checkpoint to store. If training is interrupted, we need a way to resume training from the last saved checkpoint. This can be accomplished by calling **tf.train.latest_checkpoint**, passing in the checkpoint directory. If any checkpoints exist, it will return the filepath else it returns None.

{% highlight python linenos %}
ckpt_dir = os.path.join("output", "checkpoints")

# when to start checkpoint; will be 0 when first training
start_at = 0

# Define the objects we want TF to track
ckpt_obj = tf.train.Checkpoint(
        d_opt=d_opt,
        g_opt=g_opt,
        generator=generator,
        discriminator=discriminator
)

latest_ckpt = tf.train.latest_checkpoint(ckpt_dir)

if latest_ckpt is not None:
    print("[INFO] Resuming from ckpt: {}".format(latest_ckpt))
    ckpt_obj.restore(latest_ckpt).expect_partial()
    latest_ckpt_idx = latest_ckpt.split(os.path.sep)[-1].split("-")[-1]
    start_at = int(latest_ckpt_idx)
    print(f"Resuming ckpt at {start_at}")

ckpt_callback = EpochCheckpoint(ckpt_dir, every=1, start_at=start_at, ckpt_obj=ckpt_obj)

model.fit(train_imgs, epochs=EPOCHS, callbacks=[ckpt_callback])
{% endhighlight %}

From above, we define the checkpoint directory. Next we create a **tf.train.Checkpoint** object. This allows us to define the objects we wish to track using a dictionary. In this case, we define the two optimizers and the generator and critic models. 

Next we check if we are resuming training from previous checkpoint by calling **tf.train.latest_checkpoint**. 

If this is the first time we are running the training script, there will be no checkpoints and this will return None. The script will continue to call fit and start training from scratch.

If there are any checkpoints found, it will call **restore** on the checkpoint object and attempt to extract the epoch number from its filepath. This sets the **start_at** variable which gets passed into the callback object to resume training from that specific checkpoint found.

Full working example can be found on [TF 2.0 Guide on training GAN] and the official [tf.train.Checkpoint API] has implementation examples.


Happy Hacking !!!