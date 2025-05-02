---
layout: post
show_meta: true
title: Deploying Triton Server to Sagemaker Endpoint
header: Deploying Triton Server to Sagemaker Endpoint
date: 2025-04-30 00:00:00
summary: Simple intro to deploying the Trition Server to Sagemaker Endpoint
categories: machine-learning inference trition aws sagemaker
author: Chee Yeo
---

[Installing Miniconda on Linux terminals]: https://www.anaconda.com/docs/getting-started/miniconda/install#linux-terminal-installer

[Build custom python backend]: https://github.com/triton-inference-server/python_backend#building-custom-python-backend-stub

In a previous post, I briefly described how to use Triton Server for serving an ML model for inference. The SageMaker AI service provides a hosted inference service that supports Trition Server.

In order to deploy the model with Trition using the python backend, we need to perform the following tasks:

* Create a virtualenv with the required python version and depedencies. This stage is optional. If your model doesn't require any dependecies such as opencv or numpy, the model will use the default python version in the Triton server container image. If your model requires external dependencies, you need to package them in a virtualenv and export it as an archive. The preferred and recommended approach is to use `conda`. The recommended approach is to create the conda virtualenv from within the Triton Server container image itself. I chose `23.12-py3` as it is the same image I ran on localhost for deployment. As per the recommended approaches, we run the conda env creation from within the triton container:

{% highlight shell %}
docker pull nvcr.io/nvidia/tritonserver:23.12-py3

docker run -it --rm \
-v ./model_repository:/opt/tritonserver/model_repository \
-v ./scripts:/opt/tritonserver/scripts \
nvcr.io/nvidia/tritonserver:23.12-py3
{% endhighlight %}

From within the running container, we follow the guide on [Installing Miniconda on Linux terminals]:
{% highlight shell %}
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh

bash Miniconda3-latest-Linux-x86_64.sh
{% endhighlight %}

Next, we run a custom bash script that uses Miniconda to create a virtualenv with the required dependencies. It uses `conda-pack` to package the created virtualenv into an archive file. The archive is extracted and placed within the directory of one of the models in the model repository:

{% highlight shell %}
#!/usr/bin/env bash

eval "$(/root/miniconda3/bin/conda shell.bash hook)"

conda create -y -n py310 python=3.10.12
conda init
conda activate py310
conda install -c conda-forge libstdcxx-ng=12 -y
export PYTHONNOUSERSITE=True
pip install transformers torch pillow opencv-python-headless scikit-image scipy conda-pack

conda-pack

mkdir -p model_repository/detection_postprocessing/py310
tar -xvf py310.tar.gz -C model_repository/detection_postprocessing/py310
rm -rf py310.tar.gz
{% endhighlight %}

The script above created a virtualenv of name `py310`. The python version specified must match the python version in the running container. If the versions are different, we need to [Build custom python backend].

Since the overall virtual env is large in size ( approx 5 GB ), it makes sense for it to be reused among the different models in the ensemble. I decided to place the extracted env into one of the models sub directory. We can't place the extracted virtualenv in its own directory in the model repository as Triton will fail to initialize since its looking for a config.pbtxt file in each sub-directory under the model repository.

The next step is to modify each of the config.pbtxt file for each model that uses the virtualenv. We need to add an additional parameter of `EXECUTION_ENV_PATH` which points to the location of the env. Since the model repository will be hosted on cloud storage such as S3, we can only use relative paths. AWS recommends using the env variable `$$TRITON_MODEL_DIRECTORY` which will point to the current model repository location:

{% highlight shell %}
parameters: {
  key: "EXECUTION_ENV_PATH",
  value: {string_value: "$$TRITON_MODEL_DIRECTORY/py310"}
}
{% endhighlight %}

We need to update the remaining config.pbtxt to reference the virtualenv path:

{% highlight shell %}
parameters: {
  key: "EXECUTION_ENV_PATH",
  value: {string_value: "$$TRITON_MODEL_DIRECTORY/../detection_postprocessing/py310"}
}
{% endhighlight %}


The overall directory structure of the model repository becomes:
{% highlight shell %}
model_repository/
├── detection_postprocessing
│   ├── 1
│   │   └── model.py
│   ├── py310 ( conda env with deps )
│   └── config.pbtxt
├── ensemble_model
│   ├── 1
│   └── config.pbtxt
├── grounding_dino_tiny
│   ├── 1
│   │   └── model.py
│   └── config.pbtxt
└── sam_vit
    ├── 1
    │   └── model.py
    └── config.pbtxt
{% endhighlight %}

Note that we can't place the `py310` env in its own folder within `model_repository` as triton will interpret it as a model directory and fail to start up.

We also need to remove any symlinks within the conda env as the deployment will fail with `Please ensure that the object located at the URL is a valid tar.gz archive` error. I used the following script to remove the symlinks:

{% highlight shell %}
#!/usr/bin/env bash

# Check if a directory is provided as an argument
if [ $# -eq 0 ]; then
    echo "Usage: $0 <directory>"
    exit 1
fi

# The directory to process
dir="$1"

# Find all symbolic links in the directory and its subdirectories
find "$dir" -type l | while read -r link; do
    # Get the target of the symbolic link
    target=$(readlink -f "$link")
    
    # Check if the target file exists
    if [ -e "$target" ]; then
        # Remove the symbolic link
        rm "$link"
        
        # Copy the actual file to the location of the former symbolic link
        cp -a "$target" "$link"
        
        echo "Replaced symlink: $link"
    else
        echo "Warning: Target does not exist for $link"
    fi
done
{% endhighlight %}

The next step is to create a TAR archive of the model repository. Sagemaker will upload this archive as its model files during deployment. The command below is used to create the archive locally:

{% highlight shell %}
tar --exclude="hf_cache" -cvzf ensemble_model.tar.gz -C model_repository/ detection_postprocessing ensemble_model grounding_dino_tiny sam_vit
{% endhighlight %}

Note that the name of the archive must match the name of the model. In this example, the main ensemble model name is `ensemble_model`. The command above also excludes the `hf_cache` subdirectory as they contain cached HuggingFace model files. 

Next we can prepare the Sagemaker deployment script using boto3. The following resources are required for an inference endpoint:
* Model
* Endpoint Configuration
* Endpoint
* Cloudwatch Loggroups

An IAM role is also required for Sagemaker. I created a custom `SageMakerExecutionRole` with the default policy of `AmazonSageMakerFullAccess` attached but it can be refined to restrict the permission set further.

The following is the deployment script I used:
<script src="https://gist.github.com/cheeyeo/93cab948789a5285064f146c781bc5a0.js"></script>

Line 49 tries to work out the ECR image path to download the Triton image from. AWS Deeplearning containers are organised into regions. In our case, we want to download the `sagemaker-tritonserver:23.12-py3` image for eu-west-2. You can also choose to use your own custom image but it would require setting up of Sagemaker agent to work with your model codebase, which is outside the scope of this project.

Line 52 - 55 uploads the model repository into S3 which uses the default Sagemaker bucket of `sagemaker-eu-west-2-<ACCOUNT ID>` created by Sagemaker.

We create the model first using `sm_client.create_model` to create the model. Next, we create the endpoint config object via `sm_client.create_endpoint_config` passing in the model name and the instance type. Next, we create the inference endpoint via `sm_client.create_endpoint` passing in the endpoint name and the endpoint config name. To check on the status of the deployment, we check on the endpoint status until it is no longer in `CREATING` mode.

To check the deployment status, you can inspect the deployment logs on Cloudwatch. An excerpt of a successful deployment is show below:

{% highlight shell %}
=============================
== Triton Inference Server ==
=============================
NVIDIA Release 23.12 (build <unknown>)
Triton Server Version 2.41.0
Copyright (c) 2018-2023, NVIDIA CORPORATION & AFFILIATES.  All rights reserved.
Various files include modifications (c) NVIDIA CORPORATION & AFFILIATES.  All rights reserved.
This container image and its contents are governed by the NVIDIA Deep Learning Container License.
By pulling and using the container, you accept the terms and conditions of this license:
https://developer.nvidia.com/ngc/nvidia-deep-learning-container-license
NOTE: CUDA Forward Compatibility mode ENABLED.
  Using CUDA 12.3 driver version 545.23.08 with kernel driver version 470.256.02.
  See https://docs.nvidia.com/deploy/cuda-compatibility/ for details.
I0501 13:49:20.865285 93 cache_manager.cc:480] Create CacheManager with cache_dir: '/opt/tritonserver/caches'
I0501 13:49:20.971940 93 pinned_memory_manager.cc:241] Pinned memory pool is created at '0x7f2d46000000' with size 268435456
I0501 13:49:20.972296 93 cuda_memory_manager.cc:107] CUDA memory pool is created on device 0 with size 67108864
I0501 13:49:20.974179 93 model_config_utils.cc:680] Server side auto-completed config: name: "ensemble_model"
platform: "ensemble"
input {
  name: "ensemble_image_input"
  data_type: TYPE_FP32
  dims: -1
  dims: -1
  dims: -1
  dims: -1
}
input {
  name: "ensemble_image_basename"
  data_type: TYPE_STRING
  dims: -1
}
output {
  name: "ensemble_output"
  data_type: TYPE_STRING
  dims: -1
}
ensemble_scheduling {
  step {
    model_name: "grounding_dino_tiny"
    model_version: -1
    input_map {
      key: "image_input"
      value: "ensemble_image_input"
    }
    output_map {
      key: "bounding_box"
      value: "dino_bounding_box"
    }
  }
  step {
    model_name: "sam_vit"
    model_version: -1
    input_map {
      key: "bounding_box"
      value: "dino_bounding_box"
    }
    input_map {
      key: "image"
      value: "ensemble_image_input"
    }
    output_map {
      key: "mask"
      value: "sam_mask_output"
    }
  }
  step {
    model_name: "detection_postprocessing"
    model_version: -1
    input_map {
      key: "basename"
      value: "ensemble_image_basename"
    }
    input_map {
      key: "image"
      value: "ensemble_image_input"
    }
    input_map {
      key: "mask"
      value: "sam_mask_output"
    }
    output_map {
      key: "img_path"
      value: "ensemble_output"
    }
  }
}
...
{% endhighlight %}

The inference endpoint URL can be accessed via the dashboard under SageMaker > Inference > Endpoint. It has the following format:

{% highlight shell %}
https://runtime.sagemaker.eu-west-2.amazonaws.com/endpoints/ensemble-ep-2025-05-01-13-41-30-2xl/invocations
{% endhighlight %}

Note that the endpoint can only be accessed internally within AWS sagemaker services such as from a Sagemaker Notebook. If accessing externally via boto3, you would need to create an API gateway and a lambda function with the client code as calling it directly externally will result in SSL error:

{% highlight shell %}
botocore.exceptions.SSLError: SSL validation failed for https://runtime.sagemaker.eu-west-2.amazonaws.com/endpoints/ensemble-ep-2025-04-30-20-28-14-2xl/invocations EOF occurred in violation of protocol (_ssl.c:2406)
{% endhighlight %}

Future posts will explore how to manage access to the infernece endpoint for both development and in production.

