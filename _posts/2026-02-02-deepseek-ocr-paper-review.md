---
layout: post
show_meta: true
title: Paper review of DeepSeek-OCR Contexts optical compression
header: Paper review of DeepSeek-OCR Contexts optical compression
date: 2026-02-02 00:00:00
summary: Paper review of DeepSeek-OCR Contexts optical compression
categories: paper-review deepseek-ocr
author: Chee Yeo
---

[DeepSeek-OCR: Contexts Optical Compression]: https://arxiv.org/abs/2510.18234


In the paper [DeepSeek-OCR: Contexts Optical Compression], the authors proposed a novel approach to transform long LLM contexts into images as a form of compression. The core idea is to encode long textual context as an image before sending it to an LLM in order to save on the number of textual tokens sent to an LLM as images use less tokens and can encode more textual information compared to text alone. Optical compression through the use of vision tokens can result in higher compression ratios.

DeepSeek-OCR is a new architecture that supports this approach described above. It consists of the following components:
* DeepEncoder
* Decoder

Firstly, an image input is converted into 16x16 patches, similar to how Vision Transformers work. The patches are sent as input into the encoder. The encoder is responsible for extracting image features and apply compression to visual representations.

The DeepEncoder consists of 2 main models connected sequentially:
* SAM-Base 80M model
* CLIP-Large 300M model

Between the two vision models are 2 additional convolutional layers which the paper labeled as `compressors` that performs further downsampling of the vision tokens.

The idea is that the SAM ViT model outputs vision tokens ( local / self attention ) which the `compressor` downsamples before they are sent to the CLIP model to apply global attention to, which becomes the embedding layer. The first patch embedding layer from the CLIP model is removed to achieve this.

Since the 2 layer convolutional kernel performs 16x downsampling, a single 1024x1024 image can result in only 256 tokens.

The decoder is a DeepSeek-3B model which uses a MoE architecture. During inference, the model can activate 6 out of 64 routed experts and 2 shared experts with about 570M activated parameters. The model is chosen as it has the efficiency of a small language model.

The decoder tries to reconstruct the original text representation from the compressed latent vision tokens output from the DeepEncoder and learns a non-linear mapping through OCR-style training approaches.

In terms of training, the entire architecture is trained as an end-to-end OCR model. The training dataset is carefully curated to include a mix of OCR as well as text and image data. 3 sets of data are curated as follows:
* OCR 1.0, which consists of traditional OCR tasks such as scene image and document OCR
* OCR 2.0, which includes generated charts, formulas and plane geometry with annotations
* General vision data
* Text-only data

For OCR 1.0., 




