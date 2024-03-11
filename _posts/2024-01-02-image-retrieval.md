---
layout: post
show_meta: true
title: Content-based image retrieval using large vision models
header: Content-based image retrieval using large vision models
date: 2024-01-02 00:00:00
summary: Using pretrained large vision models for content based image retrieval
categories: machine-learning torch opencv dinov2 image-retrieval
author: Chee Yeo
---

[Weaviate]: https://weaviate.io/
[DINOv2]: https://github.com/facebookresearch/dinov2
[Caltech 101 dataset]: https://data.caltech.edu/records/mzrjq-6wc02


Content-Based Image Retrieval describes a system whereby we attempt to query a datastore of image features, given an input query of an image, to find and return similar images based on its content. The image features are generated by a model which has been trained on the dataset in an autoencoder architecture. These image embeddings are retrieved from the encoder, normally before the last fully-connected layer. 

There are 2 main approaches to generating image embeddings:

* Using feature extraction, whereby we use a pretrained image model such as ResNet, and extract the image features from the layer before the FC layers.

* Using fine-tuning, whereby we remove the FC layers from the pretrained model, freeze its weights, attach a new FC layer to the model, retrain the model using a lower LR on our custom dataset.

From my own experiences, using fine-tuning yield better results but its a time consuming process as we need to tune the hyperparameters well in order for the decoder to learn to reproduce the image features properly before we can use the encoder to generate image embeddings. The entire process needs to be repeated when new images are added to the distribution.

As an experiment, I decided to try the DinoV2 model to test its effectivness in image retrieval without any fine-tuning involved.

I also wanted to try to use the Weaviate Vector database to store and retrieve image features.


### 1. Setting up Weaviate

[Weaviate] is an open-source vector database designed for AI workloads. It has pre-built modules to support text and image retrieval. In this example, we are not using the pre-built modules as we are generating our own embeddings. We use the configuration defaults as they are, which means its using the **cosine distance metric** as similarity measure.

We can run it using docker compose as follows:

{% highlight yaml %}
version: '3.4'
services:
  weaviate:
    image: semitechnologies/weaviate:1.23.0
    ports:
      - 8080:8080
      - 50051:50051
    restart: on-failure:0
    volumes:
      - weaviate_data:/var/lib/weaviate 
    environment:
      QUERY_DEFAULTS_LIMIT: 25
      AUTHENTICATION_ANONYMOUS_ACCESS_ENABLED: 'true'
      PERSISTENCE_DATA_PATH: '/var/lib/weaviate'
      DEFAULT_VECTORIZER_MODULE: 'none'
      CLUSTER_HOSTNAME: 'node1'
volumes:
  weaviate_data:
{% endhighlight %}

We attach a new docker volume to store the DB data so it persists between restarts. 

To test that its running, we can access http://localhost:8080 or use the Weaviate client in a python script:

{% highlight python %}
import os
import weaviate


WEAVIATE_URL = os.getenv('WEAVIATE_URL')
if not WEAVIATE_URL:
    WEAVIATE_URL = 'http://localhost:8080'


client = weaviate.Client(WEAVIATE_URL)
schema = client.schema.get()
print(schema)
{% endhighlight %}

Since the database is empty it should generate an empty response:
{% highlight python %}
{'classes': []}
{% endhighlight %}

We will create the DB schema next.


### 2. Create DB schema

Data is stored in Weaviate as an **object** and each object belongs to a **collection**. We can create both of them at the same time by defining its properties in a dict and passing it to the client during schema creation:

{% highlight python %}
import os
import weaviate


WEAVIATE_URL = os.getenv('WEAVIATE_URL')
if not WEAVIATE_URL:
    WEAVIATE_URL = 'http://localhost:8080'


client = weaviate.Client(WEAVIATE_URL)

class_obj = {
    "class": "Image",
    "vectorizer": "none",
    "properties": [
        {
            "name": "image",
            "dataType": ["blob"],
            "description": "image"
        },
        {
            "name": "filepath",
            "dataType": ["string"],
            "description": "filepath of image"
        }
    ]
}

client.schema.create_class(class_obj)
{% endhighlight %}

The above creates an **Image** class to store our image embeddings. It stores the **filepath** as a string and the actual image content as a **blob**, which needs to be base64-encoded before storage.

Run the above script in a separate terminal to create the schema. Once complete, we can continue to creating our image embeddings.


### 3. Generating image features

The [DINOv2] model is trained using self-supervised learning (SSL) on a specially curated dataset using a combination of SSL strategies and loss functions which makes it capable of learning image features without supervised fine-tuning.

For this example, we are using the [Caltech 101 dataset].

To obtain the image features, we first load the model in a custom class with the required preprocessing:

{% highlight python %}
import torch
import torchvision.transforms as T
from PIL import Image


class DinoV2Embed:
    def __init__(self):
        self.device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')

        self.model = torch.hub.load('facebookresearch/dinov2', 'dinov2_vits14')
        self.model.to(self.device)

        self.preprocessor = T.Compose([
            T.Resize((244, 244)),
            T.CenterCrop(224),
            T.ToTensor(),
            T.Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225])
        ])

    
    def embed(self, image_path):
        img = Image.open(image_path).convert('RGB')
        self.model.eval()

        with torch.no_grad():
            img = self.preprocessor(img)[:3].unsqueeze(0).to(self.device)

            embedding = self.model(img)
            embedding2 = embedding[0].cpu().numpy()

            return embedding2
{% endhighlight %}

The above loads the pretrained [DINOv2] model from torch hub. We pass the input image through a preprocessor that resizes it to 244, center crop it to 224 and then apply normalization. This is passed as input directly to the model. The output returned is the image embedding, which is the output of the last transformer block which has passed through LayerNormalization and is a vector of shape 384. This is important as the image embedding needs to be a vector to be stored in the vector database. 


Next, we create a custom script that can iterate over our image directory and store the embedding into our database:

{% highlight python %}
# Creates vectors if images and upload to weaviate db

from pathlib import Path
import os
import io
import base64
import weaviate
from PIL import Image
from tqdm import tqdm
from models.dinov2 import DinoV2Embed


def setup_batch(client):
    """
    Prepare batching config for Weaviate
    """

    client.batch.configure(
        batch_size=100,
        dynamic=True,
        timeout_retries=3,
        callback=None
    )


def delete_images(client):
    """
    Remove all images from vector db
    """
    with client.batch as batch:
        batch.delete_objects(
            class_name='Image',
            where={
                'operator': 'NotEqual',
                'path': ['filepath'],
                'valueString': 'x'
            },
            output='verbose'
        )



def img_to_base64(img_path):
    """
    img_content is PIL.Image ?
    """

    img = Image.open(img_path)
    img_format = img.format
    img = img.convert('RGB') # PIL.Image.Image
    img_bytes = io.BytesIO()
    img.save(img_bytes, format=img_format)
    img_bytes = img_bytes.getvalue()

    return base64.b64encode(img_bytes).decode('utf-8')


def import_data(client, source_path):
    """
    Process all images and upload its vector into db
    """
    model = DinoV2Embed()

    with client.batch as batch:
        for img_path in os.listdir(source_path):
            img_path = os.path.join(source_path, img_path)
            # print(f'IMG PATH: {img_path}')
            tqdm.write(f'IMG PATH: {img_path}')

            img_vector = model.embed(img_path)
            img_base64 = img_to_base64(img_path)

            data_properties = {
                'image': img_base64,
                'filepath': img_path
            }

            batch.add_data_object(data_properties, 'Image', vector=img_vector)


if __name__ == '__main__':
    WEAVIATE_URL = os.getenv('WEAVIATE_URL')
    if not WEAVIATE_URL:
        WEAVIATE_URL = 'http://localhost:8080'

    client = weaviate.Client(WEAVIATE_URL)

    setup_batch(client)

    delete_images(client)

    # Looks for subdir inside dataset directory
    p = Path('dataset')
    for child in tqdm(p.iterdir(), disable=None):
        tqdm.write(f'DIR: {child}')*
        import_data(client, child)
{% endhighlight %}

The above script will iterate over each subdirector in a given directory and stores the image into the database. Note that the image needs to be converted to base64 to be stored as a blob, which is the role of **img_to_base64** function. The **delete_images** function clears the database everytime this is run.


### 4. Create webapp to visualize 

To test this out, I decided to create a simple webapp using **Flask** to visualize the output of image retrieval.

> Note that the webapp doesn't have any authentication or security and is meant to be a demo. Also the model inference should be in a separate service.

{% highlight python %}
import os
from pathlib import Path
import json
from flask import Flask, render_template
from flask import request, flash, redirect, url_for
import weaviate
import sys
# add the submodules to $PATH
# sys.path[0] is the current file's path
sys.path.append(sys.path[0] + '/..')
from models.dinov2 import DinoV2Embed


app = Flask(__name__)
app.config['UPLOAD_FOLDER'] = '/tmp/uploads'
WEAVIATE_URL = os.getenv('WEAVIATE_URL', 'http://localhost:8080')


@app.route('/', methods=['GET'])
def home():
    return render_template('index.html')


@app.route('/search', methods=['POST'])
def search():
    file = request.files['file']
    # Upload file to tmp location and pass path as query to vector DB
    tmp_filename = os.path.join(app.config['UPLOAD_FOLDER'], file.filename)
    file.save(tmp_filename)

    client = weaviate.Client(WEAVIATE_URL)

    model = DinoV2Embed()
    query_vector = {
        'vector': model.embed(tmp_filename)
    }

    res = (
        client.query.get(
            'Image', ['filepath', 'image']
        )
        .with_near_vector(query_vector)
        .with_limit(10)
        .with_additional(['distance'])
        .do()
    )

    images = res['data']['Get']['Image']
    for img in images:
        img['filepath'] = '/'.join(Path(img['filepath']).parts[1:])

    return render_template('results.html', content=images)

{% endhighlight %}

The image search is within the **search** function. It takes an image upload in the browser, stores it in a temp location, and applies the [DINOv2] image embedding on it. Using the Weaviate client, it runs a custom query vector with the input image embedding. The query can be further customized with a max distance filter which can further filter out dissimilar images based on image distance computed.

Below are some screenshots of running some similarity searches.

![Search for minaret](/assets/img/cbir/results1.png)
![Search for kangaroo](/assets/img/cbir/results2.png)
![Search for cats](/assets/img/cbir/results3.png)

Based on the visual outputs alone, the embeddings produced by [DINOv2] is superior compared to using a pre-trained ResNet50 model. It's able to recognise image features at an angle as in the first example. It's also able to detect image content on its own, regardless of the background. This is not possible with a pre-trained Convnet model as it tends to pick up similar images based on image background alone.

In conclusion, it is possible to build a reliable image retrieval service using [DinoV2] with [Weaviate] database.

There is still a lot to learn on [Weaviate] database and how the [DINOv2] model works so this is left as an exercise to the reader.

H4PPY H4CK1NG