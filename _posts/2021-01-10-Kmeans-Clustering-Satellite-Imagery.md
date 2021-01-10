---
layout: post
title: KMeans Clustering for Satellite Imagery
---

Recently, I applied KMeans clustering to Satellite Imagery and was impressed by the results. I'll tell you the tricks I learned so you don't waste your time. 

![alt text](/images/kmeans/kmeans_trained_on_ea36717ca661ca3cca59d5ea43a81afc.png)

Things to note:

- Use [rasterio]() not [gdal](). Gdal [sucks](https://www.reddit.com/r/gis/comments/4tltus/a_rant_gis_software_sucks/).

- For this example I'll be using [Terravion imagery](http://www.terravion.com/). This gives high resolution low level satellite imagery. The Terravion imagery comes in 8 different [bands](https://gsp.humboldt.edu/OLM/Courses/GSP_216_Online/lesson3-1/bands.html).

-  I'll have 3 clusters. These will include:
  - Canopy cover (trees, vegetation, etc. ) 
  - Soil
  - Background 



# KMeans Explanation
I made an infographic to explain KMeans. Check it out on [reddit](https://www.reddit.com/r/learnmachinelearning/comments/kipra3/i_made_an_infographic_to_summarise_kmeans/).

Here's a small version:

<center>
<img src="https://drive.google.com/uc?id=17h24B0poX9GOywYu2xaqoMsXo3E0gLkr" align="center" width="300" />
</center>


# Stack Bands

Each Terravion image has the following bands (yours may vary):

 - `red2.tif`
 - `alpha.tif`
 - `tirs.tif`
 - `blue.tif`
 - `nir.tif`
 - `red.tif`
 - `green.tif`
 - `green2.tif`

You'll need to stack all the bands before doing anything. Here's some code for stacking the bands. It was taken from [this post](https://gis.stackexchange.com/questions/223910/using-rasterio-or-gdal-to-stack-multiple-bands-without-using-subprocess-commands).


```python
def stack_bands(files):
  img_fp = 'sentinel_bands.tif'
  # Read metadata of first file and assume all other bands are the same
  with rasterio.open(files[0]) as src0:
      meta = src0.meta

  # Update metadata to reflect the number of layers
  meta.update(count = len(files))

  # Read each layer and write it to stack
  with rasterio.open(img_fp, 'w', **meta) as dst:
      for id, layer in enumerate(files, start=1):
          with rasterio.open(layer) as srclassifer:
              dst.write_band(id, srclassifer.read(1))

  return img_fp
```

# Fit KMeans

Now we'll need to fit the KMeans classifier to the data. What I found worked best was fitting the KMeans classifier on a few images. Ideally images with a distinct difference between canopy cover and soil.

The `train` function will take in a md5 code (eg. `9bbd67cfb0d660551d74decf916b2df2`) and a date string (eg. `20190223T130237`). It will find that image on the dataset and fit the KMeans classifier on that image. 

A few important things:

- I'm not using the `alpha` and `blue` channels. They proved to be useless. 

- I'm using `red2.tif` and `green2.tif`. `red.tif` and `green.tif` didn't prove to be very useful.

Much of this training code was adapted from this [github repo](https://github.com/wathela/Sentinel2-clustering/blob/master/Sentinel2_Image_clustering.ipynb).


```python
from sklearn.cluster import KMeans
from rasterio.plot import reshape_as_image
import matplotlib.cm as cm
from sklearn import cluster

def get_date_from_text(text):
  text = text.split('/')[-2].split('_')[4]
  datetime_date = datetime.strptime(text, '%Y%m%dT%H%M%S').date()
  return datetime_date

# no alpha and blue imgs
def get_files_from_code(code, date):
  files = glob.glob('/content/canopy_cover_cotton_*'+code+'_'+date+'*TERRAV_PLN/*.tif')
  files = sorted(files, key = lambda x: get_date_from_text(x))
  files = [i for i in files if os.path.basename(i).split('.')[0] in ('red2', 'green2', 'nir', 'tirs')]
  return files

def train(k = 3, classifer = None, date = '20190223T130237', code = '9bbd67cfb0d660551d74decf916b2df2'):

  files = get_files_from_code(code, date)

  img_fp = stack_bands(files)
  img_raster = rasterio.open(img_fp)

  # Read, enhance and show the image
  img_arr = img_raster.read()
  vmin, vmax = np.nanpercentile(img_arr, (5,95))  # 5-95% contrast stretch

  # create an empty array with same dimension and data type
  imgxyb = np.empty((img_raster.height, img_raster.width, img_raster.count), img_raster.meta['dtype'])

  # loop through the raster's bands to fill the empty array
  for band in range(imgxyb.shape[2]):
      imgxyb[:,:,band] = img_raster.read(band+1)

  # convet to 1d array. 4 cause we have 4 bands here.
  img1d = imgxyb[:,:,:4].reshape((imgxyb.shape[0]*imgxyb.shape[1],imgxyb.shape[2]))
  # print(img1d.shape)

  # create an object of the classifier and train it
  if(classifer == None):
    classifer = cluster.KMeans(n_clusters=k)
  # param = cl.fit(img1d[~mask])
  param = classifer.fit(img1d)

  # get the labels of the classes and reshape it x-y-bands shape order (one band only)
  img_cl = classifer.labels_
  img_cl = img_cl.reshape(imgxyb[:,:,0].shape)

  # Show the resulting array and save it as jpg image
  plt.figure()
  plt.imshow(img_cl, cmap=cm.YlOrRd)
  plt.axis('off')
  plt.savefig("kmeans_train_image.jpg", bbox_inches='tight')
  plt.show()

  return classifer
```

```python
# train model
train_dates = ['20190502T113544', '20190502T113205']
train_codes = ['ea36717ca661ca3cca59d5ea43a81afc', '9bbd67cfb0d660551d74decf916b2df2']
model = None

for i in range(min(len(train_codes), len(train_dates))):
  model = train(classifer = model, date = train_dates[i], code = train_codes[i])
```

Here's the training output.

![alt text](/images/kmeans/training_output1.png)
![alt text](/images/kmeans/training_output2.png)

It looks pretty good!

# Prediction

Now we can run predictions on our model and see how it does. 

This function will take in the stacked bands and saved model. it will then run the KMeans model on a new image. The prediction output will be saved as a jpg file.

This saved jpg is for visualisation purposes only. Don't use it for further calculations. I accidentally did that and got very confused. 

# Create Grid

Now we can create an image grid from the saved images jpgs we obtained previously.

This makes it far easier to see if KMeans was actually correct.

```python
codes = list(set([i.split('_')[0] for i in combinations]))

for code in codes:

  k_means = glob.glob('kmeans_output/*'+str(code)+'*k_means.*g')
  k_means = sorted(k_means, key = lambda x: int(os.path.basename(x).split('.')[0].split('_')[-3]))
  k_means = [PIL.Image.open(i) for i in k_means]

  original_imgs = glob.glob('kmeans_output/*'+str(code)+'*original_image.*g')
  original_imgs = sorted(original_imgs, key = lambda x: int(os.path.basename(x).split('.')[0].split('_')[-3]))
  original_imgs = [PIL.Image.open(i) for i in original_imgs]
  
  full_list = k_means + original_imgs
  imshow_func(createImageGrid(full_list, rows = 2, scale = 0.1))
```


![alt text](/images/kmeans/kmeans_trained_on_9bbd67cfb0d660551d74decf916b2df2.png)

It does a decent job at separating canopy cover from soil. More satellite imagery would be required to comprehensively assess its performance.


# Conclusion

I did this work for a [small startup in Sydney](https://flurosat.com/). I learned so much from the experienced professionals at this startup. I couldn't have created this without their help. 

I hope this post helps someone out there. It certaintly would've helped me when I started. The full code can be found on Github

If I've made a mistake please reach out to me on [twitter](https://twitter.com/neeliyer11)

