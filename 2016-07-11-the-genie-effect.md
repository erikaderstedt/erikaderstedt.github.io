---
date: 2016-07-11 17:37:40 +0200
categories: [swift, Core Image]
---

# The Genie effect, now 100% Swiftier

Many years ago I wrote a blog post on how to recreate the classic Genie effect using a custom `CIFilter`. People keep asking for help with this, which is a bit embarrassing since the sample project no longer compiles. I'm in the process of rewriting this portion of my app and thought I'd take the opportunity to update the blog post.

<!-- more -->

First off, the Genie effect is a non-linear transformation and cannot be achieved by affine transforms. One way of doing this is to create a custom Core Image processing kernel. This kernel can then be incorporated into a `CIFilter`, which can be applied to the contents of a `CALayer`, where also animation is added.

The Genie effect consists of two phases:

* first, one side of the image contracts into a narrow strip. The opposite side remains full size.  
* then, the image moves in a fixed "envelope" from the full-size side to the narrow side.

The kernel has four input arguments: 

* the time `t`, progressing from 0 to 2. 0 to 1 is the first phase, and 1 to 2 is the second phase.
* the width `D` of the narrow strip.
* the y position (`ytarget`) of the narrow strip. This code does the effect towards the left side of the view (for animating a view into the sidebar of my app). If you need some other direction you will need to edit the kernel code.
* the scaling factor `scale` of the screen where the view is shown.

The kernel is applied for each output point and calculates the corresponding input coordinate to sample. The source code is in [this gist](https://gist.github.com/erikaderstedt/a4eba3d7f311d7aa7f0cd967d14058c7).

I instantiate this kernel lazily like this:

	let genieKernel: CIKernel = {
		guard
			let url = Bundle.main.urlForResource("ASGenieKernel", withExtension: "txt"),
			let str = try? String(contentsOf: url),
			let kernel = CIKernel(string: str) else { fatalError("Resource not accessible") }
	
		return kernel
	}()

then call it using `CIFilter.apply` in the definition for the `outputImage` property on my `CIFilter` subclass. 

To create the full animation of the view sliding to the left, I begin by taking a snapshot of the view:

	extension NSView {
		var snapshot: CGImage? {
			guard let bir = self.bitmapImageRepForCachingDisplay(in: self.bounds) else {
				NSLog("No snapshot available."); return nil }
			
			self.cacheDisplay(in: self.bounds, to: bir)
			
			guard let inputImage = bir.cgImage else {
				NSLog("Could not get cgImage from cached image."); return nil }
	
			return inputImage
		}
	}
 	
I then create a layer of the same size at the view, add it to the view's layer, and set the snapshot as the `contents` for the the layer. I create an instance of the `CIFilter` subclass and assign it to the `filters` property. Finally, I create an animation over the time property on the filter, going from 0 to 2, and wrap the animation in a `CATransaction` to remove the layer once the animation completes.

The rest of the code can be found in [this gist](https://gist.github.com/erikaderstedt/8257477c84f855bd67cf6dc45f129b77).

