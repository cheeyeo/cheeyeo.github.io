---
layout: post
show_meta: true
title: Introduction to Triton Server
header: Introduction to Triton Server
date: 2025-04-24 00:00:00
summary: Simple intro to the Trition Server for inference
categories: machine-learning inference trition
author: Chee Yeo
---

[Nvidia Triton Server]: https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/introduction/index.html

In a previous post, I discussed building a pipeline for processing receipt images. While the setup works for local development, it is not easy to deploy as it makes use of 2 different Huggingface models in addition to custom post-processing opencv code. An approach would be to wrap it around a framework such as pytorch but the issue of how to version and deploy models still poses an issue. 

One solution I have found was to use [Nvidia Triton Server], which is designed for deploying ML models including LLMs. It's an OTS solution with the following features:

* Support for various ML frameworks such as TF, Pytorch, scikitlearn means that you can run models built using these different frameworks and they can be accessed in the same way via the same client scrips.

* Supports different backend types such as Onnx, pytorch and even custom python code.

* Allows you to build model pipelines with support for Ensembles or Business Logic Scripting for more complex workflows.

* Supports HTTP and GRPC protocols for client inference.

* Built-in metrics endpoint for sending metrics to Prometheus for example.

For my use case, I identitied the key components for migration to Triton:

* Grounding-DINO HF model
* SAM HF model
* Custom OpenCV code

I decided to use the **python** backend as it allows me to wrap the HF models and custom logic into python classes which can be chained together to form an ensemble. This would support a single inference call which would simplify the workflow.

The core concept behind trition is the use of a model repository. The model repository is storage which holds all the models you want to serve via Trition. On startup, the server would traverse every single sub-directory in this path and registers every model that has the right configuration file. Even if the backend differs between each model but the configuration is valid, they will still be served by Trition.

The model repository needs to have the following structure for each model:

```
/model_repository/
├── detection_postprocessing
│   ├── 1
│   │   └── model.py
│   └── config.pbtxt
├── grounding_dino_tiny
│   ├── 1
│   │   └── model.py
│   └── config.pbtxt
└── sam_vit
    ├── 1
    │   └── model.py
    └── config.pbtxt
```

From the above output, the model repository contains 3 active models. Each model hosted has a **version number** as a folder which contains the model code. The default policy is to serve the latest model version which has the highest numerical value. One can also specify a certain version to use via the configuration file. The **config.pbtxt** specifies the configuration to apply to the model such as its name, backend type, and inputs and outputs data types and shape. For example, the config.pbtxt for grounding_dino_tiny model is as follows:

```
name: "grounding_dino_tiny"
backend: "python"

parameters: {
    key: "huggingface_model",
    value: {string_value: "IDEA-Research/grounding-dino-tiny"}
}

parameters {
    key: "box_threshold",
    value: {string_value: "0.3"}
}

parameters {
    key: "text_threshold",
    value: {string_value: "0.25"}
}

input [
    {
        name: "image_input"
        data_type: TYPE_FP32
        dims: [ -1, -1, -1, -1 ]
    }
]

output [
    {
        name: "bounding_box"
        data_type: TYPE_INT32
        dims: [ -1, -1 ]
    }
]
```

The backend field is set to **python** as we are wrapping the HF model within a custom python class which will be explained later. The model takes 3 parameters which are passed via the `model_config` object into the model class when instantiated. Note that parameter values can only be of string types. Next, we declare that the model has a single input of **image_input** of FP32 ( floating point 32 ) type and of shape [ batch_size, channel, height, width ]. A single output of INT32 ( integer 32 ) of shape [ batch_size, array ] is returned which contains the bounding box coordinates. The other models also have similar configuration files.

The use of inputs and outputs allow us to pass data between the client and the server but also between different models via the client code. In our use case, we need to perform the following steps with Triton:

* Send an image to ground-dino-tiny model to obtain bounnding-box coordinates
* If they exist, send them as inputs to the sam-model for obtain a segmentation mask
* If it exists, send the image and mask to the detection postprocessing model to obtain the final artifact

While this works, this is not an efficient use of Trition as it would involve making 3 separate API calls to the inference server. A better approach would be to use ensembles, which would only involce 1 API call per image when implemented. This will be discussed in a future post.

To send inference requests to the server, we need to create Triton clients using the libraries. Firstly, we need to install the client lib:

```
pip install tritonclient[all]
```

Next, we instantiate an instance of **httpclient.InferenceServerClient** to send the inference request. The example below shows the initial client request for grounding dino model:

```
client = httpclient.InferenceServerClient(url="localhost:8000")
image = np.asarray(Image.open("receipt1.jpg")).astype(np.float32)

image = np.expand_dims(image, axis=0)

input_tensors = [httpclient.InferInput("image_input", image.shape, datatype="FP32")]
input_tensors[0].set_data_from_numpy(image)

outputs = [httpclient.InferRequestedOutput("bounding_box")]

query_response = client.infer(
    model_name="grounding_dino_tiny",
    inputs=input_tensors,
    outputs=outputs
)

print(query_response.as_numpy("bounding_box")) # [[ 611 1514 1718 3111]]
```

To pass an input to the server, we need an instance of **httpclient.InferInput** which states the input name; the input data shape and the data type. From the earlier config.pbtxt file, we can see that the inputs values are specified in the infer input object directly. The outputs are defined by a **httpclient.InferRequestedOutput** object which contains the output name. Both the inputs and outputs are passed as parameters to the **infer** action. Any respnse can be obtained by calling **as_numpy** with the output name as the parameter.

( TODO: Show running triton docker image )


If the client call is successful, the results will be returned. If there are any issues with the model code, it will be reported via the terminal and the error will be in the logs for the client code which also indicate where the failure is in the model code. 