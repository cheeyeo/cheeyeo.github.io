---
layout: post
show_meta: true
title: Learning about Transformers with semantic segmentation
header: Learning about Transformers with semantic segmentation
date: 2023-07-01 00:00:00
summary: Learning about transformers by training a segformer model
categories: machine-learning python pytorch google-colab segformer transformer
author: Chee Yeo
---

[Segformer research paper]: https://arxiv.org/abs/2105.15203

In order to understand more about **attention** and the **transformer** architecure, I set out to discover other uses of transformers in computer vision and through online learning courses, I came across the **Segformer** model, which is a well-known model used for segmantic segmentation. Semantic segmentation is the process of assigning each pixel of an image to a given class.

The **Segformer** model uses a encoder-decoder architecture. The encoder comprises of **MixTransformer** blocks, which uses a form of attention mechanism known as **Efficient Self Attention**, to learn a representation of the underlying data at various resolutions/scales e.g. 1/4, 1/8, 1/16. The decoder comprises of MLP layers which combines these learned multiscale features from the encoder, in effect, combining both local and global attention, resulting in rendering more powerful representations. Unlike Vit, Segfomer doesn't use positional encoding. A detailed walkthrough of the segformer model architecture is beyond the scope of this post and readers are encouraged to read the [Segformer research paper] to discover more.

The initial model built from the course, was based on B0 variant of the Segformer model. However, the provided training weights were based on the B3 variant of the model. I created my own version of the B3 model based on the original paper by:

* Tuning the hyperparameters to match that of the original paper.

* Using the original full version of the Cityscape training data at 1024x2048 resolution.

* Creating custom modules to support distributed data parallel training; logging; model checkpoints; data preprocessing; data augmentation

A single input with model parameters require at least 9GB of GPU memory. My inital attempt at training the model locally yield poor results as I was only able to train with a batch size of 1. The paper recommended a batch size of 8. I was able to gain access to a A100 GPU via Colab Pro which allowed me to train using a batch size of 4. Subsequent blog posts will detail multi-gpu, multi-node training on custom GCP VMs.

The following details the changes I made to improve the original code.


### Changes to dataloader

The Dataloader code has to be changed to accept **DistributedSampler**. **pin_memory** is set to True, and **num_workers** specified to match CPU cores. torch uses a **default_collate** function for dataloaders which converts each mini-batch into Tensors of the right data format. However, in my experiments, I have to use the custom **pseudo_collate** function from **mmengine** in order to overcome the exception of `RuntimeError: Trying to resize storage that is not resizable` during training

{% highlight python %}
if train is False:
  # For validation/test use distributed sampler
  sampler = DistributedSampler(dataset, world_size, rank, shuffle=shuffle)
else:
  # For training use infinite sampler
  sampler = InfiniteSampler(dataset,
                            shuffle,
                            random_seed)

dataloader = DataLoader(dataset,
                        batch_size=batch_size,
                        sampler=sampler,
                        num_workers=num_workers,
                        pin_memory=True,
                        shuffle=shuffle,
                        collate_fn=pseudo_collate,
                        drop_last=drop_last,)
{% endhighlight %}

During training via iterations, we need the dataloader to continously return mini-batches of data since the number of iterations is greater than the actual dataset size. We use a **InfiniteSampler** from **mmsegmentation** project which loops back to the beginning when we exceed the dataset size but still within the training loop.


### Changes to training loop

The initial model uses epoch-based training. I modified it to fit the paper's description of iteration-based training. 

An epoch makes a single pass across the entire train dataset via mini-batches. Iteration based training makes a single pass over a single mini-batch of data. Hence the need to use an iterator-based sampler as above. 

The training loop becomes:

{% highlight python %}
min_val_loss = 0.0

dataloader_iterator = InfiniteDataLoaderIterator(dataloader_train)

model.train()
for iteration in range(start_iter, max_iter):
  data_batch = next(dataloader_iterator)
  inputs, labels = data_batch
  inputs, labels = data_pre({"inputs": inputs, "labels": labels}, training=True)

    y_preds = model(inputs)
    loss = criterion(y_preds, labels)

    optimizer.zero_grad()
    loss.backward()
    optimizer.step()

    linear_scheduler.step()
    polynomial_scheduler.step()

    train_loss = loss.mean()

    if world_rank == 0:
      # Print train loss
    
    if (iteration % eval_every == 0):
      # Run evaluation at intervals
      validation_loss = evaluate_model()

      if validation_loss < min_val_loss and (world_rank==0):
        # If this is the new min val loss we save the model
        min_val_loss = validation_loss
        model_state_dict = model.module.state_dict()
        state = {
          'state_dict': model_state_dict
        }

        torch.save(state, "model.pt")

    
    if (save_checkpoint and world_rank == 0):
      # Create state dict
      state = {
        'iteration': iteration,
        'state_dict': model.state_dict(),
        'optimizer': optimizer.state_dict(),
        'scheduler1': scheduler1.state_dict(),
        'scheduler2': scheduler2.state_dict()
      }
      torch.save(state, "checkpoint.pt")
{% endhighlight %}

Within the training loop, we get a mini-batch of training data using a custom class of **InfiniteDataLoader**. This class wraps the distributed data loader as input and calls **__next__** on the underlying iterator. It resets the data to the beginning once it reaches a **StopIteration** exception which is handled via the **InfiniteSampler** object automatically.

We wrap the inputs and labels in another preprocessing object which essentially reformats the data into the right shape by stacking and place it on the right device for training.

We make a forward pass, compute the loss and perform backprop by calling `loss.backward()` and `optimizer.step()`. Note we also need to call step on the two learning rate schedulers from previous discussion.

Note the use of **world_rank**. This value is set automatically when running distributed data parallel training and indicates which device the training is currently occuring. In a multi-gpu environment, we only want to log and save model state on the master device, in this case of world rank 0.

The following steps indicate that if we are on device 0:
* Print out the train loss

* Run validation and only if we are on device 0, if the validation loss has decreased, we save the model's weights

* Check if we need to save model checkpoint but only if its running on device 0.


### Changes to the learning rate scheduler

The paper described using a learning rate schedule that linearly increase the learning rate from a lower value of 1e-6 to 6e-5 for the first 1500 iterations. A polynomial learning rate scheduler then decays the learning rate from 6e-5 from 1500 iteration onwards. This was implemented via **OneCycleLR** in the initial code but was unstable in the training process.

This is now implemented via 2 different learning rate schedulers, **LinearLR** and **PolynomialLR** during training. The state of these schedulers are also saved as part of the checkpoint during training as shown above. During training, we perform backprop and call `step` method of each scheduler individually as above.


### Changes to trainer script

My initial idea for the training script was to have an implementation that could run in various environment from single gpu instances to multi-node, multi-gpus instances. 

I decided to run the training script via `torchrun` on single instances and `mpirun` on multi-nodes instances since these two processes set environment variables on the world and local rank which we could use to pass to the training loop.

{% highlight python %}
# other imports omitted for brevity
import os
import sys
import torch.distributed as dist
from torch.nn.parallel import DistributedDataParallel as DDP
...

if 'LOCAL_RANK' in os.environ:
    # Environment variables set by torch.distributed.launch or torchrun
    LOCAL_RANK = int(os.environ['LOCAL_RANK'])
    WORLD_SIZE = int(os.environ['WORLD_SIZE'])
    WORLD_RANK = int(os.environ['RANK'])
elif 'OMPI_COMM_WORLD_LOCAL_RANK' in os.environ:
    # Environment variables set by mpirun
    LOCAL_RANK = int(os.environ['OMPI_COMM_WORLD_LOCAL_RANK'])
    WORLD_SIZE = int(os.environ['OMPI_COMM_WORLD_SIZE'])
    WORLD_RANK = int(os.environ['OMPI_COMM_WORLD_RANK'])
    
    # Setting batch size eql to 1 means in DDP, if we have 8 GPUS we have a global batch size of 1 * 8
else:
    sys.exit("Can't find the environment variables for local rank")
{% endhighlight %}

The above checks for the presence of certain environment variables such as **LOCAL_RANK** which is set via `torchrun`. We then retrieve the relevant information such as world size and local rank. We apply the same concept to check for `mpirun` and set those same values accordingly. If none of those are true, we exit with an exception.

The trainer function wraps the model in a **DistributedDataParallel** object and initializes the training process:

{% highlight python %}
def wrap_model():
  # other code omitted for brevity...
  set_random_seeds(1234, cudnn_deterministic=True, cudnn_benchmark=False)

  torch.distributed.init_process_group(backend="nccl", rank=WORLD_RANK, world_size=WORLD_SIZE) 

  # build dataloaders

  # Init model and move to device
  model = SegFormerMITB3(in_channels=3, num_classes=19, embed_dim=256)
  model = model.to(LOCAL_RANK)

  model.decoder_head = torch.nn.SyncBatchNorm.convert_sync_batchnorm(model.decoder_head)

  ddp_model = DDP(model, device_ids=[LOCAL_RANK], output_device=LOCAL_RANK)

  if LOCAL_RANK == 0:
    map_location = {"cuda:0": "cuda:{}".format(LOCAL_RANK)}

    if resume_path is not None:
      ckpt = torch.load(resume_path, map_location=map_location)
      ddp_model.load_state_dict(ckpt['state_dict'])
      # Load the state for the optimizer, scheduler
    else:
      # Load the pretrained weights only
      ddp_model.module.backbone.load_state_dict(torch.load(pretrained_weights, map_location=map_location))
  

  # Additional check for multi-gpu, multi-node setup
  # Load the same checkpoint for the other gpus 

  if WORLD_SIZE > 1:
    dist.barrier()

    if LOCAL_RANK > 0:
      map_location = {"cuda:0": "cuda:{}".format(LOCAL_RANK)}

      if resume_path is not None:
        ckpt = torch.load(resume_path, map_location=map_location)
        ddp_model.load_state_dict(ckpt['state_dict'])
        # Load the state for the optimizer, scheduler
      else:
        # Load the pretrained weights only
        ddp_model.module.backbone.load_state_dict(torch.load(pretrained_weights, map_location=map_location))
{% endhighlight %}

The above sets the random seeds for distributed training. It calls `torch.distributed.init_process_group` passing NCCL as the backend and setting the world rank and size information.

It initializes a model instance and moves it to the local device via **LOCAL_RANK**. Next it wraps the model in a **DistributedDataParallel** object passing in the local rank information.

The remaining section checks for the presence of a **resume_path** variable which specifies the checkpoint to resume training from. If set, it loads the model checkpoint for device 0 first. Next, it checks whether we are running in a multi-gpu environment. If so, we set a **dist.barrier()** for synchronization so that we can safely load the checkpoint into the other models on the other devices before resuming training. 


### Training in Colab

My initial attempts at using Colab involved saving the dataset and model artifacts in google drive and mounting the drive into the notebook. This resulted in issues with having to monitor and manage usage on the drive which is inefficient.

Subsequently, I migrated the dataset into a GCP bucket and mount it into the notebook during training. 

However, I ran into the issue of slow training per iteration as the model was trying to read each batch of data from the mounted bucket. I resolved this by copying the data from the bucket to the **/content** path of the notebook. I created a second bucket to store training artifacts such as checkpoints and logs. I mounted the second bucket as a local path into **/content** using **gcsfuse**. 

{% highlight bash %}
# Install fuse and gcsfuse to mount gcp buckets
!echo "deb https://packages.cloud.google.com/apt gcsfuse-focal main" | sudo tee /etc/apt/sources.list.d/gcsfuse.list
!curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
!sudo apt-get update
!sudo apt install fuse gcsfuse
!gcsfuse -v
!sudo apt install tree


from google.colab import auth

auth.authenticate_user()

!mkdir -p /content/mount
!gcsfuse --implicit-dirs modeltraining /content/mount

!mkdir -p /content/dataset

project_id = 'PROJECT_ID'
!gcloud config set project {project_id}
!gsutil ls
!gsutil -m cp -r gs://cityscape_data/* /content/dataset/
!tree --dirsfirst --filelimit 10 /content/dataset
{% endhighlight %}

I started the training script as follows:
{% highlight bash %}
!torchrun \
--nproc_per_node=1 --nnodes=1 --node_rank=0 \
--master_addr=localhost --master_port=1234 \
train_segformers.py \
--batch_size 4 --logs /content/mount/logs/"$(date +"%d-%m-%Y")" --ckpts /content/mount/ckpts/"$(date +"%d-%m-%Y")" --artifacts /content/mount/artifacts/"$(date +"%d-%m-%Y")" --dataset /content/dataset
{% endhighlight %}

The above runs the training script via `torchrun`. Since we are running on a single GPU instance, we set the number of processes and nodes to be 1 and the rank to be 0. Next, we invoke the actual script which accepts arguments that specify the location of log files and checkpoint storage paths. These paths map to the mounted bucket **modeltraining** from above. Any files saved to these paths are automatically synced to the bucket storage.

There are still some improvements that can be made to the training process such as using mixed-precision training and using the new torch built-in transformers features to reduce attention weights. These will be in future posts.


### References

* [SegFormer: Simple and Efficient Design for Semantic Segmentation with Transformers](https://arxiv.org/abs/2105.15203)
