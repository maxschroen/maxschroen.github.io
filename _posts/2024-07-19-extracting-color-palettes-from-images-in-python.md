---
title: "Extracting Color Palettes from Images in Python"
last_modified_at: 2024-07-19T018:35:16+01:00
classes: wide
categories:
  - Tutorials
tags:
  - Python
  - Clustering
  - Showcase
  - Tutorial
---
This short tutorial showcases a fairly straightforward solution to a problem I have encountered while working on a recent project of mine. For my album artwork poster generation script I was looking to include a feature that displays 5 colored blobs representing 5 colors from the album cover art as a visual flair like in below example. 

![album-poster-example](/assets/images/posts/color-palettes/misery-signals-of-malice-and-the-magnum-heart.jpg){: width="400" style="display:block; margin-left:auto; margin-right:auto"}
<p align="center" style="font-size:small;">Example Album Poster with Colored Blobs</p>

The simplest way to do this would be to just pick the desired amount of colors at random from the image. And while this would technically satisfy the requirement, it would be a rather unscientific solution. The visual fidelity of the results from this method would also suffer greatly since we are working with inherently lossy, compressed images. These images in general but especially around sharp edges, e.g. from contrasting colors, contain more compression artefacts that would falsify the result by picking colors not present in the original image.

![compression-artefact-example](/assets/images/posts/color-palettes/compression-artifact-example.png){: width="500" style="display:block; margin-left:auto; margin-right:auto"}
<p align="center" style="font-size:small;">Compression Artefact Example (<a href="https://www.researchgate.net/publication/334360686_Fully_Convolutional_Network_for_Removing_DCT_Artefacts_From_Images">Source</a>)</p>

A more appropriate and scientific way to accomplish our goal is to extract a representative color palette from the image, which can be achieved via a variety of algorithms. Most commonly used for this task are the median cut and k-means clustering algorithms. Both of these make use of the dimensional properties of color values in the RGB space, which can be represented as 3-dimensional vectors along their red, green and blue color component dimensions. While the median cut algorithm performs exceptionally well at color quantization tasks, retaining a larger visual fidelity at reduced color depth when compared to k-means, I personally have found the results produced by this algorithm to be too closely grouped and muted, whereas k-means produces more distinct and saturated color variations. Therefore I have decided to use k-means for the project.

Since we are working with RGB images, the image data is can be portrayed by a $m$ x $n$ matrix with $n$ being the pixel width and $m$ the pixel height of the image. Every pixel is represented by a 3-dimensional vector containing the color channel information for red, green and blue. This concept is shown in the graphic below, but in the visualization the color channel values have been normalized to fit $[0,1]$ whereas they normally would be within $[0..255]$.

![pixel-array](/assets/images/posts/color-palettes/pixel-array.jpg){: width="400" style="display:block; margin-left:auto; margin-right:auto"}
<p align="center" style="font-size:small;">RGB Image Pixel Array (<a href="https://levelup.gitconnected.com/pixels-arrays-and-images-ef3f03638fe7">Source</a>)</p>

To retrieve this information from our image, we first read the image data via [pillow](https://pillow.readthedocs.io/en/stable/), a fork of the Python image manipulation (PIL) library. After retrieving the pixel data, we reshape the 2-dimensional array into a 1-dimensional array since we do not care about the location of the pixels in the image, only the color values. With the reshaped array we now have the input data for the clustering algorithm.

```python
from PIL import Image
import numpy as np

# Replace with your image path
image_path = './path/to/your/image.jpg'
# Get 2D pixel data array
image = Image.open(image_path) 

# Convert to ndarray
image_array = np.array(image)
# Reshape the image array to a 1D array of RGB vectors
pixels = image_array.reshape(-1, 3) 
```

<p align="center" style="font-size:small;">Code Sample: Extracting Pixel Color Data</p>

We now use the array of RGB vectors for our k-means clustering. The algorithm works by placing k random points into the data domain, they act as our initial set of cluster centroids. All data points are then associated to their spacially closest centroid, forming k clusters. The new cluster centroid location is then calculated as the mean along all dimensions of the cluster elements. With the updated centroids, the previous 2 steps of assignment and centroid update are repeated until the centroids no longer significantly move after the update. This is called convergence. After the clusters converge, we can have our final result in the form of the last available cluster assignments for each point. In below animation this is visualized on a 2-dimensional random example dataset.

![k-means-animation](/assets/images/posts/color-palettes/k-means-animation.gif){: width="450" style="display:block; margin-left:auto; margin-right:auto"}
<p align="center" style="font-size:small;">K-Means Clustering Animation (<a href="https://code-specialist.com/python/k-means-algorithm">Source</a>)</p>

Implementing this algorithm yourself is a great way to challenge yourself and get a deeper understanding of its mechanics and algorithmic coding. For now however, we do not have to reinvent the wheel to apply this algorithm to our already extracted pixel data and can make use of the [scikit-learn](https://scikit-learn.org/stable/) library, which already comes with a fully implemented class for k-means.

```python
from sklearn.cluster import KMeans

# Set number of colors to extract
num_colors = 5
# Fit the KMeans model to the pixel data
kmeans = KMeans(n_clusters=num_colors) 
kmeans.fit(pixels)
# Get the values of the centroids    
colors = kmeans.cluster_centers_.astype(np.uint8)
```
<p align="center" style="font-size:small;">Code Sample: Applying K-Means Algorithm</p>

Finally, we get an unordered array of cluster centroid values in the form of RGB color arrays representing color palette extracted from the image color data.

```python
colors
>>> array([[140,  73,  22],
           [114, 122, 146],
           [ 63,  26,   6],
           [ 69,  84, 119],
           [195, 177, 157]], dtype=uint8)
```
<p align="center" style="font-size:small;">K-Means Output Containing RGB Colors</p>

Displaying this next to our sample image, we can see that we get a fairly well distributed representation of colors from the image despite our cluster count $(k=5)$ being relatively low. We get a good representation of the gradient of earthy tones in the foreground and distance as well as a light and dark blue hue for the sky. Increasing the cluster count would yield more nuanced results.

![color-palette](/assets/images/posts/color-palettes/color-palette.jpg){: width="450" style="display:block; margin-left:auto; margin-right:auto"}
<p align="center" style="font-size:small;">Sample Image (<a href="https://unsplash.com/photos/a-field-with-a-mountain-in-the-background-r_wAAUeXDcQ">Source</a>) with Extracted Color Palette</p>

Since the count of individual pixels in an image can become quite large fairly quickly, especially at higher modern resolutions(4k @ 16:9 &rarr; ~9.4m px; 8k @ 16:9 &rarr; ~37,7m px), this will increase the runtime and required space of our clustering significantly. A good mitigation at the cost of some accuracy is to downsize the image either completely before use in the script or directly via Python prior to clustering. Here it is important to account for the aspect ratio of the image, so the color proportions are preserved relative to the original image. Firstly, this of course does not apply for square images, as there applies $width = height$ and the resizing can be done without any additional calculations. Secondly, we also need to keep in mind that this step can be skipped altogether if the image is already small enough, as otherwise in case $width_{target} < width_{original}$ the image would be upsized instead.

```python
# Define target width
target_width, target_height = 512, None
# Check if image width is less than target width
if image.width < target_width:
    # Check for square images
    if image.height == image.width:
        # Square image dimensions
        target_height = target_width
    else:
        # Calculate reduction ratio between original and target width
        reduction_ratio = float(target_width / image.width)
        # Calculate target height by applying reduction rate
        target_height = int(image.height * reduction_ratio)
    # Resize image to target dimensions
    image = image.resize((target_width, target_height))
```
<p align="center" style="font-size:small;">Code Sample: Image Resizing</p>

As a last additional step, we can take care of the fact that the clustering results returned from the k-means class are unordered. This brings us to the consideration of how colors can even be ordered in a meaningful way. One possibility would be to sum all color components from the RGB channels in order to create a ranking on brightness. And while this naive approach would work in a strictly mathematical sense in most cases, it would be ignoring the anatomy of the human eye that is most sensitive to green, less sensitive to red and least sensitive to blue light. To account for this physiological effect, in photometry the measure of [_relative luminance_](https://en.wikipedia.org/wiki/Relative_luminance) $Y$, sometimes also referred to as perceived brightness, is used. This measure is calculated by converting the standard RGB values ($[0..255]$) to linear RGB ($[0,1]$) and applying scaling factors to each color component based on the human eye's sensitivity to the respective color component and summing the results, like so: 

$$Y = 0.2126 * R_{lin} + 0.7152 * G_{lin} + 0.0722 * B_{lin}$$

<p align="center" style="font-size:small;">Relative Luminance Formula (<a href="https://en.wikipedia.org/wiki/Relative_luminance">Source</a>)</p>

To apply this relative luminance score as a ranking measure in ascending order, we can use the following code:

```python
def luminance(color: list) -> float:
    # Define color component luminance weights
    LUMINANCE_WEIGHTS = [0.2126, 0.7152, 0.0722]
    # Calculate luminance score
    luminance = sum(LUMINANCE_WEIGHTS[i] * color[i] for i in range(len(color))) / 255
    return luminance

# Sort existing color palette by relative luminance
colors_sorted = [color for color in sorted(colors, key=lambda color: luminance(color))]
```

<p align="center" style="font-size:small;">Code Sample: Ordering Colors by Relative Luminance</p>

From the ranked results, we can see that although very similar for this specific case, the perceptually brighter light brown color is correctly ranked as brighter than the perceptually darker dark blue hue when using the luminance ranking approach as compared to the naive approach. With the low number of colors this result is less pronounced and more noticeable with a larger number of extracted colors, but the shown results will suffice for this showcase.

![color-comparison](/assets/images/posts/color-palettes/color_comparison.png){: width="400" style="display:block; margin-left:auto; margin-right:auto"}
<p align="center" style="font-size:small;">Luminance vs. Naive Color Rankings</p>

Lastly, below there is the complete code for this small showcase, taking care of opening and resizing the image to a manageable resolution, then applying the k-means clustering algorithm and ordering the resulting extracted color palette by the relative luminance of the contained colors.

```python
from PIL import Image
import numpy as np
from sklearn.cluster import KMeans

def luminance(color: list) -> float:
    # Define color component luminance weights
    LUMINANCE_WEIGHTS = [0.2126, 0.7152, 0.0722]
    # Calculate luminance score
    luminance = sum(LUMINANCE_WEIGHTS[i] * color[i] for i in range(len(color))) / 255
    return luminance

# Replace with your image path
image_path = './path/to/your/image.jpg'
# Get 2D pixel data array
image = Image.open(image_path) 

# Define target width
target_width, target_height = 512, None
# Check if image width is less than target width
if image.width < target_width:
    # Check for square images
    if image.height == image.width:
        # Square image dimensions
        target_height = target_width
    else:
        # Calculate reduction ratio between original and target width
        reduction_ratio = float(target_width / image.width)
        # Calculate target height by applying reduction rate
        target_height = int(image.height * reduction_ratio)
    # Resize image to target dimensions
    image = image.resize((target_width, target_height))

# Convert to ndarray
image_array = np.array(image)
# Reshape the image array to a 1D array of RGB vectors
pixels = image_array.reshape(-1, 3) 

# Set number of colors to extract
num_colors = 5
# Fit the KMeans model to the pixel data
kmeans = KMeans(n_clusters=num_colors) 
kmeans.fit(pixels)
# Get the RGB color values of the cluster centers          
colors = kmeans.cluster_centers_.astype(np.uint8)

# Sort colors by relative luminance
colors_sorted = [color for color in sorted(colors, key=lambda color: luminance(color))]
```

<p align="center" style="font-size:small;">Full Showcase Code</p>

---

The project utilizing this code can be accessed via below GitHub repository. It aims to be a configurable and modular open-source print-it-yourself alternative to commercially offered print services for album artwork posters, that is easily accessible via an accompanying CLI tool. Thanks for checking it out! 

<p align="center">
  <a href="https://github.com/maxschroen/artworker">
    <img src="https://gh-card.dev/repos/maxschroen/artworker.png?fullname="/>
  </a>
</p>

<!-- SCRIPTS -->
<script id="MathJax-script" async src="https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.5/latest.js?config=TeX-MML-AM_CHTML"></script>
<script type="text/x-mathjax-config">
   MathJax.Hub.Config({
     extensions: ["tex2jax.js"],
     jax: ["input/TeX", "output/HTML-CSS"],
     tex2jax: {
       inlineMath: [ ['$','$'], ["\\(","\\)"] ],
       displayMath: [ ['$$','$$'], ["\\[","\\]"] ],
       processEscapes: true
     },
     "HTML-CSS": { availableFonts: ["TeX"] }
   });
</script>