---
layout: post
title:  Detectron2 Web App 
---

__Building a Web App in Docker and Flask for Detectron2__

(show video here)

Detectron2 offers state of the art instance segmentation models. It's very [quick to train](https://detectron2.readthedocs.io/notes/benchmarks.html) and offers very good results. 


Model training is is fairly straightforward. There are many [tutorials](https://github.com/facebookresearch/detectron2/blob/master/GETTING_STARTED.md) to help you there. Deploying the model to a web app is a different story. In this post we'll create a web app for detectron2's instance segmentation. 

# Backend

First, we'll create the machine learning backend. This will use basic [flask](https://flask.palletsprojects.com/en/1.1.x/). We'll start from some fairly standard [boilerplate code](https://github.com/realpython/flask-boilerplate/blob/master/app.py).

```python
import io
from flask import Flask, render_template, request, send_from_directory, send_file
from PIL import Image
import requests
import os
import urllib.request

app = Flask(__name__)

@app.route("/")
def index():

	# render the index.html template
	return render_template('index.html')

if __name__ == "__main__":

	# get port. Default to 8080
	port = int(os.environ.get('PORT', 8080))

	# run app
	app.run(host='0.0.0.0', port=port)

```
This app will simply render the template `index.html`. I've specified the port manually. 

Next we'll add functions to get the image. We want to be able to upload an image to the website. We also want to be able to supply the website with a url and the image will be downloaded automatically. I've created the code do to exactly that below.

```python
import io
from flask import Flask, render_template, request, send_from_directory, send_file
from PIL import Image
import requests
import os

# function to load img from url
def load_image_url(url):
	response = requests.get(url)
	img = Image.open(io.BytesIO(response.content))
	return img

@app.route("/detect", methods=['POST', 'GET'])
def upload():
	if request.method == 'POST':

		try:

			# open image
			file = Image.open(request.files['file'].stream)

			# remove alpha channel
			rgb_im = file.convert('RGB')
			rgb_im.save('file.jpg')
		
		# failure
		except:

			return render_template("failure.html")

	elif request.method == 'GET':

		# get url
		url = request.args.get("url")

		# save
		try:
			# save image as jpg
			# urllib.request.urlretrieve(url, 'file.jpg')
			rgb_im = load_image_url(url)
			rgb_im = rgb_im.convert('RGB')
			rgb_im.save('file.jpg')

		# failure
		except:
			return render_template("failure.html")

	return send_file(rgb_im, mimetype='image/jpeg')

```
This piece of code allows us to upload an image into the backend (POST request). Or we can supply the backend with a url and it will download the image automatically (GET request). The code also converts the image to a `jpg`. I couldn't do inference on a `png` image using detectron2. So we'll have to convert to a `jpg`.

If the code can't download the image for whatever reason - It will return the `failure.html` template. This will basically just be a simple `html` page saying there was an error in retrieving the image. 

Also, I've specified a different `@app.route` (/detect). This will need to refelcted in the `index.html` file. 

# Frontend

Now I'll create the frontend `html` code. Through this inferface the user can upload an image, and also specify a url to the image. 

```
<!DOCTYPE html>
<html lang="en">

<body>

<h1 style="text-align:center;">Detectron2 Web App</h1>
<br>
<h2>Detectron2 Instance Segmentation</h2>

<form action = "/detect" method = "POST" enctype = "multipart/form-data">
	<input type = "file" name = "file" />
	<input name = "submit" type = "submit"/>
</form>
<form action = "/detect" method = "GET" enctype = "multipart/form-data">
	<input type="text" name="url">
	<input type = "submit"/>
</form>
```

There's not much to it. We create a simple form and tell it to link to the `app.route('/detect')` flask code. We also need to specify the method. If the user is uploading an image, it's POST. If the user is giving us the url to an image, it's GET. 

The `failure.html` template is even simpler. 

```
{% block content %}
<body>

    <p> Error in retrieving image </p>

</body>
{% endblock %}
```

Now we can move on the actual deep learning part. 

# The Model

In this part, we'll get a detectron2 pretrained model to do inference on an image. Then we'll link it to our existing backend. 

This part is slightly more involved. We'll create a new class called `Detector`. In that we'll create the `cfg` that is required for detectron2. Then we'll create another function that will do the inference on an image. 

I'll be using the mask rcnn pretrained model trained on the [Imagenet](http://www.image-net.org/) dataset. It will use a [ResNet](https://towardsdatascience.com/an-overview-of-resnet-and-its-variants-5281e2f56035)+[FPN](https://towardsdatascience.com/review-fpn-feature-pyramid-network-object-detection-262fc7482610) backbone. This model is said to obtain the [best speed/accuracy tradeoff](https://github.com/facebookresearch/detectron2/blob/master/MODEL_ZOO.md#common-settings-for-coco-models). This model is trained on a 3x schedule (~37 COCO epochs).

You'll have to download the model from [here](https://dl.fbaipublicfiles.com/detectron2/COCO-InstanceSegmentation/mask_rcnn_R_50_FPN_3x/137849600/model_final_f10217.pkl). 


```python
import cv2 as cv
import json
from detectron2.engine import DefaultPredictor
from detectron2.config import get_cfg
from detectron2.utils.visualizer import Visualizer
from detectron2.utils.visualizer import ColorMode
from detectron2 import model_zoo
from detectron2.data import MetadataCatalog, DatasetCatalog
from detectron2.modeling import build_model
import torch
import numpy as np
from PIL import Image

class Detector:

	def __init__(self):

		# set model and test set
		self.model = 'mask_rcnn_R_50_FPN_3x.yaml'

		# obtain detectron2's default config
		self.cfg = get_cfg() 

		# load values from a file
		# self.cfg.merge_from_file("test.yaml")
		self.cfg.merge_from_file(model_zoo.get_config_file("COCO-InstanceSegmentation/"+self.model)) 

		# set device to cpu
		self.cfg.MODEL.DEVICE = "cpu"

		# get weights 
		# self.cfg.MODEL.WEIGHTS = model_zoo.get_checkpoint_url("COCO-InstanceSegmentation/"+self.model) 
		self.cfg.MODEL.WEIGHTS = "model_final_f10217.pkl"

		# set the testing threshold for this model
		self.cfg.MODEL.ROI_HEADS.SCORE_THRESH_TEST = 0.7  

		# build model from weights
		# self.cfg.MODEL.WEIGHTS = self.convert_model_for_inference()

	# build model and convert for inference
	def convert_model_for_inference(self):

		# build model
		model = build_model(self.cfg)

		# save as checkpoint
		torch.save(model.state_dict(), 'checkpoint.pth')

		# return path to inference model
		return 'checkpoint.pth'

	# detectron model
	# adapted from detectron2 colab notebook: https://colab.research.google.com/drive/16jcaJoc6bCFAQ96jDe2HwtXj7BMD_-m5	
	def inference(self, file):

		predictor = DefaultPredictor(self.cfg)
		im = cv.imread(file)
		outputs = predictor(im)

		# with open(self.curr_dir+'/data.txt', 'w') as fp:
		# 	json.dump(outputs['instances'], fp)
		# 	# json.dump(cfg.dump(), fp)

		# get metadata
		metadata = MetadataCatalog.get(self.cfg.DATASETS.TRAIN[0])

		# visualise
		v = Visualizer(im[:, :, ::-1], metadata=metadata, scale=1.2)
		v = v.draw_instance_predictions(outputs["instances"].to("cpu"))

		# get image 
		img = Image.fromarray(np.uint8(v.get_image()[:, :, ::-1]))

		# write to jpg
		# cv.imwrite('img.jpg',v.get_image())

		return img

```

This piece of code pretty much does everything we need it to do for inference. We just need to specify the path to the pretrained model that we downloaded. 

It gets the config that corresponds our pretrained model automatically from the model zoo. It should also get the pretrained model from the model zoo as well. But I found that this doesn't really work in docker - at least for me. 

The next step is to integrate this `Detector` class into our existing script.

```python
from ObjectDetector import Detector
import io
from flask import Flask, render_template, request, send_from_directory, send_file
from PIL import Image
import requests
import os

app = Flask(__name__)
detector = Detector()


# function to load img from url
def load_image_url(url):
	response = requests.get(url)
	img = Image.open(io.BytesIO(response.content))
	return img


# run inference using detectron2
def run_inference(img_path = 'file.jpg'):

	# run inference using detectron2
	result_img = detector.inference(img_path)

	# clean up
	try:
		os.remove(img_path)
	except:
		pass

	return result_img


@app.route("/")
def index():
	return render_template('index.html')


@app.route("/detect", methods=['POST', 'GET'])
def upload():
	if request.method == 'POST':

		try:

			# open image
			file = Image.open(request.files['file'].stream)

			# remove alpha channel
			rgb_im = file.convert('RGB')
			rgb_im.save('file.jpg')
		
		# failure
		except:

			return render_template("failure.html")

	elif request.method == 'GET':

		# get url
		url = request.args.get("url")

		# save
		try:
			# save image as jpg
			# urllib.request.urlretrieve(url, 'file.jpg')
			rgb_im = load_image_url(url)
			rgb_im = rgb_im.convert('RGB')
			rgb_im.save('file.jpg')

		# failure
		except:
			return render_template("failure.html")


	# run inference
	# result_img = run_inference_transform()
	result_img = run_inference('file.jpg')

	# create file-object in memory
	file_object = io.BytesIO()

	# write PNG in file-object
	result_img.save(file_object, 'PNG')

	# move to beginning of file so `send_file()` it will read from start    
	file_object.seek(0)

	return send_file(file_object, mimetype='image/PNG')


if __name__ == "__main__":

	# get port. Default to 8080
	port = int(os.environ.get('PORT', 8080))

	# run app
	app.run(host='0.0.0.0', port=port)

```

In this piece of code I've added in the `Detector` class we created previously. I've also created a function called `run_inference`. This is where the backend will run the detectron2 model on an image. It will take in an image path and call the detectron2 model through the `Detector` class we created earlier.

The `run_inference` function will return an image once instance segmentation is completed. I had to do some hacky stuff with the result fo the `run_inference` function to get it to work. `result_img` is pasted into a file object and that file object is returned as a png. There might be a better way to do this though. 


# Docker

The final step is creating a docker container for our code. Then we'll deploy the docker container locally. 

Thankfully, detectron2 has already created a [dockerfile](https://github.com/facebookresearch/detectron2/blob/master/docker/Dockerfile) for us. So we can work from that code. 

```docker
# adapted from: https://github.com/facebookresearch/detectron2/blob/master/docker/Dockerfile

FROM nvidia/cuda:10.2-cudnn7-devel-ubuntu18.04

ENV DEBIAN_FRONTEND noninteractive
RUN apt-get update && apt-get install -y \
	python3-opencv ca-certificates python3-dev git wget sudo curl && \
  rm -rf /var/lib/apt/lists/*

# create a non-root user
ARG USER_ID=1000
RUN useradd -m --no-log-init --system  --uid ${USER_ID} appuser -g sudo
RUN echo '%sudo ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers
USER appuser
WORKDIR /home/appuser

ENV PATH="/home/appuser/.local/bin:${PATH}"
RUN wget https://bootstrap.pypa.io/get-pip.py && \
	python3 get-pip.py --user && \
	rm get-pip.py

# install dependencies
# See https://pytorch.org/ for other options if you use a different version of CUDA
RUN pip install --user tensorboard
RUN pip install --user torch==1.5 torchvision==0.6 -f https://download.pytorch.org/whl/cu101/torch_stable.html

RUN pip install --user 'git+https://github.com/facebookresearch/fvcore'
# install detectron2
RUN git clone https://github.com/facebookresearch/detectron2 detectron2_repo
# set FORCE_CUDA because during `docker build` cuda is not accessible
ENV FORCE_CUDA="1"
# This will by default build detectron2 for all common cuda architectures and take a lot more time,
# because inside `docker build`, there is no way to tell which architecture will be used.
ARG TORCH_CUDA_ARCH_LIST="Kepler;Kepler+Tesla;Maxwell;Maxwell+Tegra;Pascal;Volta;Turing"
ENV TORCH_CUDA_ARCH_LIST="${TORCH_CUDA_ARCH_LIST}"

RUN pip install --user -e detectron2_repo

# add dir
COPY requirements.txt /home/appuser/detectron2_repo
RUN pip install --user -r /home/appuser/detectron2_repo/requirements.txt
RUN pip install --user 'git+https://github.com/cocodataset/cocoapi.git#subdirectory=PythonAPI'


# Set a fixed model cache directory.
ENV FVCORE_CACHE="/tmp"
WORKDIR /home/appuser/detectron2_repo
ENV PILLOW_VERSION=7.0.0

# add dir
COPY . /home/appuser/detectron2_repo

# Make port 8080 available to the world outside the container
ENV PORT 8080
EXPOSE 8080

CMD ["python3", "app.py"] 

```

I've bascially used the dockerfile supplied in detectron2's [github repo](https://github.com/facebookresearch/detectron2/blob/master/docker/Dockerfile). But I made a few changes. 

I've added a `requirements.txt` [file](insert link here later). I do a `pip install` from that requirements file. That installs a few libaries that we need for this to work. I've also changed the command to start the `app.py` script we created earlier. That will start the flask application and render the `index.html` template. 


Now we can start the docker container. We can do this using the following:

```
docker build . -f Dockerfile -t detectron2 &&\
docker run -d -p 8080:8080 detectron2
```

That will build the current dockerfile in the current working directory. Then it will run that image on port 8080. That's the port that I specified earlier. 

But the issue with that if you keep running those commands over time you'll have too many docker images and containers on your computer. This can take up a [lot of space](https://jimhoskins.com/2013/07/27/remove-untagged-docker-images.html) on your computer.

[Jim Hoskins](https://jimhoskins.com/2013/07/27/remove-untagged-docker-images.html) has a pretty elegant solution to this, which I adapted for my purposes. 

I created a nice shell script which:
 - stops all the containers
 - removes untagged containers
 - builds a new container
 - runs the container on port 8080
 - shows the trailing logs for the container

This script is *incredibly* useful. 

```shell
# stop containers
docker stop $(docker ps -a -q) 

# remove containers
docker rm $(docker ps -a -q) && docker rmi $(docker images | grep '^<none>' | awk '{print $3}')

# build container
docker build . -f Dockerfile -t detectron2

# run contrainer on port 8080
docker run -d -p 8080:8080 detectron2

# see logs
docker logs -f -t $(docker ps -q)
```

Run this script from terminal. It will build and run the detectron2 web app. The web app should show up at `localhost:8080` on your browser.

![alt text](/images/detectron2_web_app/detectron2_web_app.png)
![alt text](/images/detectron2_web_app/detect.jpeg)

It works great!

The code for this can be found on [github](https://github.com/spiyer99/detectron2_web_app). 


