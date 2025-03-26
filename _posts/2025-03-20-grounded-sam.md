---
layout: post
show_meta: true
title: Review of Grounded-SAM paper and its applications
header: Review of Grounded-SAM paper and its applications
date: 2025-03-20 00:00:00
summary: Review of Grounded-SAM paper and its applications
categories: machine-learning computer-vision sam dino
author: Chee Yeo
---

[Grounded SAM: Assembling Open-World Models for Diverse Tasks]: https://arxiv.org/abs/2401.14159

In the paper [Grounded SAM: Assembling Open-World Models for Diverse Tasks], the authors present an architecture that combined two expert models: SAM ( Segment Anything Model ) and Grounding DINO to automate image segmentation tasks. 

The Grounding-DINO model is capable of zero-shot detection via text prompts and return a set of bounding box coordinates. The SAM model is capable of masking any object in an image with prompts such as points, bounding box coordinates, or text.

Given an image and a text prompt of objects to look for, we can send it to the Grounding DINO model to generate bounding boxes. These bounding boxes can be used by SAM downstreaam to generate masks of the objects from within the image, which we can use to segment the object from the image.

The paper provided several applications of this architecture which include:

* Automatic detection and segmentation of objects via user inputs ( Grounding-DINO -> SAM ).

* Automatic image annotation via image caption model such as RAM, using their outputs ( captions, tags ) as inputs to generate bounding boxes and masks for each instance.

* Locate specific regions of interest in an image via text prompt to Grounding-SAM which passes the generated masks to an image generation model such as Stable-Diffusion to further edit the image, such as direct replacements or to generate new images as training data.

* Integration with object tracking via models such as DEVA to perform object tracking based on user text prompts.

Through my own experimentation for a project, I found an approach of using Grounded-SAM to perform OCR on receipt images, which I will detail below.


### Using Grounded-SAM for OCR

One of the projects I have been working on involve applying OCR on receipt images to track expenditure. The initial pipeline involves a series of processing steps using computer vision techniques such as thresholding and perspective transform to process a receipt image into an OCR ready image. A sample image from the dataset is shown below.

![Original Image](/assets/img/grounded-sam/orig11.jpg)


To obtain reliable OCR results, the processed image needs to be in a scanned document layout with the text against a white background, from a top-down perspective. In addition, the computer vision techniques applied for processing need to work consistenly across the different receipt images.

These are some of the issues I encountered while using the traditional computer vision approach:

* Each retailer uses a different font and layout for their receipts. Trying to create a single configuration to process the different variants require a lot of trial and error and manual testing to achieve the desired results.

* Receipts could be damaged with a corner missing or crumpled. This causes the computer vision processing to fail as its unable to detect the four corners necessary to create a top-down transform perspective.

* Receipts could be taken against a coloured background. This could cause the algorithms such as thresholding to pick up stray pixels, causing the transformation process to fail.

I refactored the pipeline to use Grounded-SAM as follows:

* Initial uploaded image gets processed by Grounding-DINO with a single text label of `receipt.` This allows the model to detect and return bounding box coordinates of the object in the image.

* The bounding box coordinates are sent to the SAM model, which creates a mask of the detected object.

* The mask is passed into an OpenCV algorithm which applies bitwise operations to remove the receipt using the mask image. The segmented object is converted to grayscale and thresholding is applied to binarize the image. 

* The binarized image has a four-point transform applied to it to create a top-down perspective.

* The image is converted into a black-and-white scanned document format using Sauvola thresholding.

* The processed imags is then rotated to the correct orientation and any skew in its text is detected and corrected.

Below is an extract of the OpenCV processing code used:

{% highlight python linenos %}
    mask = mask.transpose(1, 2, 0)
    mask = np.array(mask*255).astype("uint8")
    mask = cv2.cvtColor(mask, cv2.COLOR_GRAY2BGR)

    segmented_image = mask
    segmented_image = cv2.bitwise_or(segmented_image, mask)
    segmented_image = cv2.bitwise_and(segmented_image, np.asarray(image))
    segmented_image2 = cv2.cvtColor(segmented_image, cv2.COLOR_BGR2GRAY)
    
    ret, thresh = cv2.threshold(segmented_image2, 50, 255, cv2.THRESH_BINARY)

    contours, _ = cv2.findContours(thresh, cv2.RETR_TREE, cv2.CHAIN_APPROX_SIMPLE)
    largest_contours = sorted(contours, key=cv2.contourArea, reverse=True)[:10]
    receipt_contour = get_receipt_contour(largest_contours)

    scanned = wrap_perspective(np.asarray(image.copy()), contour_to_rect(receipt_contour))
    scanned = bw_scanner22(scanned)

    angle = determine_skew(scanned)
    rotated = rotate(scanned, angle, (255, 255, 255))

    saved_img_path = os.path.join(Path.cwd(), "processed", f"processed_{basename}")
    
    cv2.imwrite(saved_img_path, rotated)
{% endhighlight %}

The example below shows the sample image after processing:

![Processed Image](/assets/img/grounded-sam/processed1.jpg)

This end-to-end approach means that we no longer have to adjust the thresholding or any of the computer vision parameters as the input image will always be a single receipt set against a black background. 

This means that we have more visibility and explanablity into each stage of the transformation process. We are also able to process a wider range of receipts since we are not tied down to specific OpenCV algorithms and its parameters.

However, data from the real world is always messy and imperfect. The pipeline will be unable to capture the entire receipt object from a given image if for example, the receipt itself is damaged with missing corners or if the text has faded. This would result in processed images with a skew in its text. Below is an example of an input image with folded corners and the transformation resulted in the processed image to have a skew in its text:

![Processed Image](/assets/img/grounded-sam/test.jpg)

However, this could be fixed by applying text skew correction using OpenCV as discussed in my previous post:
https://www.cheeyeo.xyz/aws/textract/ocr/machine-learning/computer-vision/2024/05/22/notes-on-textract/

Once we have the OCR image, we can send it to the `AWS Textract` service which could extract the text of the purchase made using the analyze expense feature. The extracted attributes include:

* vendor's name
* date and time of purchase
* total amount spent
* each line item with its corresponding quantity, description and price

The receipt data is stored in a MongoDB database for further analysis.

A simple web application is also created using python Flask to visualize the results.

![Processed Image](/assets/img/grounded-sam/webapp.png)

In addition, this approach also allows us to replace the `AWS Textract` service with another such as tesseract or a similar cloud service. The extracted data could also be used for creating a training dataset to train another OCR model such as `DonutOCR`.

By using Grounded-SAM to process OCR, we are able to automate the workflow of processing receipt images using an ensemble of models each with a specific task, making the process less error-prone. The processing is performed in a sequential manner in different stages, which lends itself to using a state machine or a DAG pipeline to monitor and track each step of the process.