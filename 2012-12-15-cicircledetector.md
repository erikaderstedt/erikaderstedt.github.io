---
date: 2012-12-15 17:49
categories: [Cocoa, Image analysis]
---

# CICircleDetector

Let's say that you allow the user to load an orienteering map in the form of a scan or a photo, and you want to know where the control circles are. You can

* ask the user
* or figure it out yourself.

This post is how you would go about figuring it out yourself.
<!-- more -->
![Original map](http://www.aderstedt.net/blog_images/1.png "Original map")

As with any image analysis problem, the first step is to determine what additional information we have available. In this case:

* the colour of the circle is well known. Although it can vary a bit due to printer / scanner / camera issues, colour is very important to orienteering maps and tightly controlled.
* the size of the circle is not known, but it is a circle (and not an approximation thereof).
* all circles are the same size.
* there may be gaps in the circle line (to show important detail underneath)
* there may be a few lines connecting to the circle (but not too many)
* there are no other objects inside the circle of the same colour (except when two circles overlap).
* the circles are fully inside the map

Colour filtering
----------------

Given that the colour is the same, I filter out anything with this colour. This is a two-step Core Image filter. First I rotate the hue so that purple resides only in a single channel:

	// Hue adjustment and filtering
	CIFilter *hueAdjust = [CIFilter filterWithName:@"CIHueAdjust"];
	[hueAdjust setDefaults];
	[hueAdjust setValue:@(45.0/180.0) forKey:@"inputAngle"];
	[hueAdjust setValue:sourceImage forKey:@"inputImage"];

Then I filter on this channel:

	CIFilter *matrix = [CIFilter filterWithName:@"CIColorMatrix"];
	[matrix setDefaults];
	[matrix setValue:[CIVector vectorWithX:1.0 Y:-1.0 Z:-1.0 W:0.0] forKey:@"inputRVector"];
	[matrix setValue:[CIVector vectorWithX:1.0 Y:-1.0 Z:-1.0 W:0.0] forKey:@"inputGVector"];
	[matrix setValue:[CIVector vectorWithX:1.0 Y:-1.0 Z:-1.0 W:0.0] forKey:@"inputBVector"];
	[matrix setValue:[CIVector vectorWithX:0.0 Y:0.0 Z:0.0 W:1.0] forKey:@"inputAVector"];
	[matrix setValue:[hueAdjust outputImage] forKey:@"inputImage"];

Quartz Composer is a very useful tool for quickly determining what will work and what won't work. 

Circle detection
----------------

Now I have an image like this:

![Colour-adjusted](http://www.aderstedt.net/blog_images/2.png "Colour-adjusted")

Now, when I see something like this, I think of convolution filters. I actually tried this, but 

* it is quite slow, given that the filter needs to encompass the entire circle
* a convolution filter cannot use all the information that we have available to us.

The former can possibly be addressed by using Fourier transforms to do the convolution, but the latter cannot. What I ended up with doing was checking the neighbourhood of each pixel in the image for a circular area without any coloured pixels, a thin circular area outside it where more than half of the pixels were coloured, and finally an area outside of that area where most pixels weren't coloured. 

This calculation needs to be done for each source pixel. To reduce the computational load, I scale the image down. The primary source of images will be the camera on a recent iOS device, and these are overkill resolution-wise for what I need. The actual iteration can and should be done in parallel. I've experimented with a few different stride lengths, and found that using dispatch_apply on each line in the image yielded the best results.

During the calculation I convert the image to a planar (single channel) floating point image. I actually use a vImage buffer for this purpose, as I had originally intended to use some of the functions in the Accelerate framework for the heavy lifting. In the end I didn't, but still kept it in this format rather than rolling my own which would have ended up essentially the same.

One convenient optimization uses the last bit of information that I have on the circles: they are located slightly away from the border of the map. This lets me crop the area of the image that we run the calculation on. As a result, I

* reduce calculation times
* don't need to handle cases where I'd need to access areas outside of the image buffer.

As previously mentioned, the size of the circles isn't known. I loop over a reasonable range of circle sizes and pick the one that gives the highest filter response. Unfortunately, the response is quite sensitive to the circle size (the wrong circle size yields a very low response), which means that I cannot use any of Householder's methods effectively. 

The result
----------

Finally, this method may well lead to multiple hits on several pixels at the center of a circle. To get the best one, we require each hit to be a local maximum in a radius equal to the circle radius. After doing that we get the results below. Hits are indicated by black dots.

![Result](http://www.aderstedt.net/blog_images/3.png "The result")

Although you can't tell from the cropped image above, there are a few missed circles, and (fewer) false positive. This performance level is consistent across quite a few maps. For the next step of the project I'm going to try to fit a GPS track including split-times to the map. I'm hoping that this performance will be enough. 
