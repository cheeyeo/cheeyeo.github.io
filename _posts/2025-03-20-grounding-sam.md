---
layout: post
show_meta: true
title: Review of Grounding SAM and its applications
header: Review of Grounding SAM and its applications
date: 2025-03-17 00:00:00
summary: Review of Grounding SAM and its applications
categories: machine-learning computer-vision sam dino
author: Chee Yeo
---

[Grounded SAM: Assembling Open-World Models for Diverse Tasks]: linktopaper

In the paper [Grounded SAM: Assembling Open-World Models for Diverse Tasks], the authors present an architecture that combined two expert models, SAM ( Segment Anything Model ) and Grounding DINO to automate image segmentation tasks. 

The Grounding-DINO model is capable of zero-shot detection via text prompts and return its bounding box coordinates. SAM model is capable of masking any object in an image with prompts such as points, bounding box coordinates, or text.

Given an image and a text prompt of objects to look for, we can send it to the Grounding DINO model to generate bounding boxes. These bounding boxes can be used by SAM downstreaam to generate masks of the objects from within the image, which we can use to segment the object from the image.

The paper provided several applications of this architecture which include:

* Automatic detection and segmentation of objects via user inputs ( Grounding-DINO -> SAM ).

* Automatic image annotation via image caption model such as RAM, using their outputs ( captions, tags ) as inputs to generate bounding boxes and masks for each instance.

* Locate specific regions of interest in an image via text prompt to Grounding-SAM which passes the generated masks to an image generation model such as Stable-Diffusion to further edit the image, such as direct replacements or to generate new images as training data.

* Integration with object tracking via models such as DEVA to perform object tracking based on user text prompts.

Through my own experimentation for a project, I found an approach of using the Grounded-SAM to perform OCR on receipt images, which I will detail below.


### Using Grounded-SAM for OCR

One of the projects I have been working on involve applying OCR on receipt images to track expenditure. The initial pipeline involves a series of processing steps using computer vision techniques such as thresholding and perspective transform to process the receipt image into an OCR ready image. A sample image from the dataset is shown below.

( sample receipt image )


To obtain reliable OCR results, the processed image needs to be in a scanned document layout with the text against a white background, from a top-down perspective. 

These are some of the issues I encountered while using this approach:

* Each retailer uses a different font and layout for their receipts. Trying to create a single configuration to process the different variants require a lot of trial and error and manual testing to achieve the desired results.

* Receipts could be damaged with a corner missing or crumpled. This causes the computer vision processing to fail as its unable to detect the four corners necessary to create a top-down transform perspective.

* Receipts could be taken against a coloured background. This could cause the algorithms such as thresholding to pick up stray pixels, causing the transformation process to fail.

I refactored the pipeline to use Grounded-SAM as follows:

* Initial uploaded image gets processed by Grounding-DINO with a single text label of `receipt.` This allows the model to detect and return bounding box coordinates of the object in the image.

* The bounding box coordinates are sent to the SAM model, which creates a mask of the detected object.

* The mask is passed into an OpenCV algorithm which applies bitwise operations to remove the receipt using the mask. The segmented object is converted to grayscale and thresholding is applied to binarize the image. 

* The binarized image has a four-point transform applied to it to create a top-down perspective.

* The image is converted into a black-and-white scanned document format using Sauvola thresholding.

* The processed imags is then rotated to the correct orientation and any skew in its text is detected and corrected.

( example of pre and post processed images )

This end-to-end apporach means that we are no longer dealing with having to adjust the thresholding or any of the computer vision parameters as the input image will always be a single receipt set against a black background. 

This means that we have more visibility and explanablity into each stage of the transformation process. We are also able to process a wider range of receipts since we are not tied down to specific OpenCV parameters.

The downside of it is it's not always able to capture the entire receipt image properly as the original input image could have corners missing. This would result in processed images with a skew in its text. This may or may not be fixed by the deskew process.

( example of deskewed image )


Once we have the OCR image, we can send it to the `AWS Textract` service which could extract the text of the purchase made using the analyze expense feature. The extracted attributes include:

* vendor's name
* date and time of purchase
* total amount spent
* each line item with its corresponding quantity, description and price

The receipt data is stored in a MongoDB database for future analysis.

A simple web application is also created to visualize the results.

( show screenshot of web app )

In addition, this approach also allows us to replace the `AWS Textract` service with another such as using tesseract or another similar cloud service. The extracted data could also be used for creating a training dataset to train another OCR model such as DonutOCR.

By using Grounded-SAM to process OCR, we are able to automate the workflow of processing receipt images using an ensemble of models each with a specific task, making the process less error-prone. The processing is performed in a sequential manner in different stages, which lends itself to using a state machine or a DAG pipeline to monitor and track each step of the process.