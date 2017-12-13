Introduction to Depth Data Screencast Script

[TODO Faraz: Add metadata at top of each script, according to writing guide phase 3]

## Talking Head

Hey what’s up everybody, this is Faraz. In this video, I’m going to show you how you can create filters to alter photos using depth data. 

The iPhone 7+, 8 and 8+, and the iPhone X have a dual camera system which allows  you to capture depth information when taking pictures in portrait mode. This is similar to depth data captured from an Xbox Kinect’s infared sensors, except without the infared. 

“This screencast is based on a tutorial created by team member Yono Mittlefehldt
 If you like this screencast, be sure to give him a follow on twitter.”

Alright, now lets implement some filters using depth data. 

## Demo

[Demo begins with starter project running on the screen]

I have a starter project here that shows some images, and you can tap to cycle between the images. It also has some buttons at the bottom to switch between different visual representations of depth data. They don’t do anything yet – that’s what we’re going to create together.

Let’s start with coding a visual representation for the depth in an image. We’re going to make it such that the closer an object is, the whiter it is, and the farther it is, the darker it will be.

You generally use AVDepthData to extract this auxiliary data from an image, so that’s the first step.


Open DepthReader.swift, we will start by adding a method to DepthReader to obtain the depth data map from an image:
 

First, you get a URL for an image file and safely type cast it to a CFURL.
You then create a CGImageSource from this file.
From the image source at index 0, you copy the disparity data from its auxiliary data, we’ll touch on disparity later. The index is 0 because there is only one image in the image source. iOS knows how to extract the data from JPGs and HEIC files alike, but unfortunately this doesn’t work in the simulator.
You prepare a property for the depth data. As previously mentioned, you use AVDepthData to extract the auxiliary data from an image.
You create an AVDepthData entity from the auxiliary data you read in.
You ensure the depth data is the the format you need: 32-bit floating point disparity information.
Finally, you return this depth data map.

```
func depthDataMap() -> CVPixelBuffer? {
  guard let fileURL = Bundle.main.url(forResource: name, withExtension: ext) as CFURL? else {
    return nil
  }

  // 2
  guard let source = CGImageSourceCreateWithURL(fileURL, nil) else {
    return nil
  }

  // 3
  guard let auxDataInfo = CGImageSourceCopyAuxiliaryDataInfoAtIndex(source, 0, 
      kCGImageAuxiliaryDataTypeDisparity) as? [AnyHashable : Any] else {
    return nil
  }

  // 4
  var depthData: AVDepthData

  do {
    // 5
    depthData = try AVDepthData(fromDictionaryRepresentation: auxDataInfo)

  } catch {
    return nil
  }

  // 6
  if depthData.depthDataType != kCVPixelFormatType_DisparityFloat32 {
    depthData = depthData.converting(toDepthDataType: kCVPixelFormatType_DisparityFloat32)
  }

  // 7
  return depthData.depthDataMap
}
```



Now before you can run this, you need to update DepthImageViewController.swift.


Find the load current image, with extension method

Create a DepthReader entity using the current image.
Using your new depthDataMap method, you read the depth data into a CVPixelBuffer.
You then normalize the depth data using a provided extension to CVPixelBuffer. This makes sure all the pixels are between 0.0 and 1.0, where 0.0 are the furthest pixels and 1.0 are the nearest pixels.
You then convert the depth data to a CIImage and then a UIImage and save it to a property.

```
// 1
let depthReader = DepthReader(name: name, ext: ext)

// 2
let depthDataMap = depthReader.depthDataMap()

// 3
depthDataMap?.normalize()

// 4
let ciImage = CIImage(cvPixelBuffer: depthDataMap)
depthDataMapImage = UIImage(ciImage: ciImage)
``` 
  
Great! We’ve now created our first filter! 
Now if you run this, you should see the depth within the photo represented in grayscale with white objects being close to you and dark objects being far away. 
 
I recommend running this on a device, as the simulator seems to be extremely slow at processing these filters. 
 
## Talking Head 

[TODO Faraz: make slides for this talking section and any other talknig section that can benefit. You can re-use the illustrations from the tutorial.]

In a nutshell, the iPhone’s dual cameras are imitating stereoscopic vision.

Try this. Hold your index finger closely in front of your nose and pointing upward. Close your left eye. Without moving your finger or head, simultaneously open your left eye and close your right eye.

The closer an object is to your eyes, the larger the change in its relative position compared to the background. Does this sound familiar? It’s a parallax effect!

Now quickly switch back and forth closing one eye and opening the other. Pay attention to the relative location of your finger to objects in the background. See how your finger seems to make large jumps left and right compared to objects further away?

The iPhone’s dual cameras are like its eyes, looking at two images taken at a slight offset from one another. It corresponds features in the two images and calculates how many pixels they have moved. This change in pixels is called disparity.
 
The farther an object is, the smaller the disparity but the larger the depth. 
Depth is in fact the inverse of disparity and is equal to one over the disparity. 

You’re going to use this slider, along with the depth data, to make a mask for the image at a certain depth. Then you’ll use this mask to filter the original image and create a neat effect!

How will we do this? Well, first, it's helpful to think of the depth data map as the following function:

[Show slide for pixel value vs. disparity here]

The pixel value of your depth map image is equal to the normalized disparity. Remember, that a pixel value of 1.0 is white and a disparity value of 1.0 is the closest to the camera. On the other side of the scale, a pixel value of 0.0 is black and a disparity value of 0.0 is furthest from the camera.

We want to create a function that uses the depth in the image to convert the image to greyscale. The closer the pixel is, the whiter it will be, and the farther away it is, the darker it will be.

However, if we simply converted the image this way as we did previously, it would be boring. So we're going to spice things up and create a filter function that focusses on a specific depth of the image so that we can use a slider to change which part of the image is in focus for the filter. 

[TODO Faraz: Add missing section from tutorial about "Using a slope of 4.0, a width of 0.1, and 0.75 as the focal point.." and slide to go along with it here.]

The pixels around the focal point will be white and will ramp down to black around that depth. Let's see how this works.

## Demo

Open up DepthImageFilters.swift and find the createMask method with focus and scale. 

Before we can convert the depth data into an image mask, we need to define some constants. 

These help to create our filter function based on the slope of our mask parameters, filter width, and focus. 

s1 is the slope of our function that defines how fast we ramp up from a black pixel to a white pixel towards the focal point. 

s2 is just the inverse of slope 1 for ramping back down after the focal point. 

The filter width is the entire length of the filter. Since we are only focussing on a particular focal point, this includes the ramp up, focal point, and ramp down. 

Note that to make the focal section a little more distinguishable, instead of only making a single point completely white, we make the pixels at that point white for a certain width (which is indicated by MaskParams.width). 

We will separate the filter function into two masks, ramping up and ramping down.

b1 is used to control where the ramping up mask is, and b2 is used to move where the ramping down mask is. 
 
```
let s1 = MaskParams.slope
let s2 = -MaskParams.slope
let filterWidth =  2 / MaskParams.slope + MaskParams.width
let b1 = -s1 * (focus - filterWidth / 2)
let b2 = -s2 * (focus + filterWidth / 2)
```

Now we want to take the constants we created to input them into the first mask (mask0) which ramps up pixels before the focal point towards white. 

Let's take our depth image and apply a filter to it. We want to base the greyscale values to be based on the slope that ramps up to white (s1). Let's apply that to the red, green, and blue vectors of the mask. 

Now we want to translate this function based on this slope, the focal point, and the width of the function. Luckily we already calculated this as b1. Let's input this into the bias vector of the mask.

Now we want to apply a clamp on the colour so that we don't have values above 1 or below 0.  


```
let mask0 = depthImage
  .applyingFilter("CIColorMatrix", parameters: [
    "inputRVector": CIVector(x: s1, y: 0, z: 0, w: 0),
    "inputGVector": CIVector(x: 0, y: s1, z: 0, w: 0),
    "inputBVector": CIVector(x: 0, y: 0, z: s1, w: 0),
    "inputBiasVector": CIVector(x: b1, y: b1, z: b1, w: 0)])
  .applyingFilter("CIColorClamp")
```


Now we want to input our constants into the other side of our function which applies a ramping down mask and applies it to the depth image.
We want to do the same thing we did before, however now using s2 to ramp down and b2 to move the second slope to where it needs to be. 

```
let mask1 = depthImage
  .applyingFilter("CIColorMatrix", parameters: [
    "inputRVector": CIVector(x: s2, y: 0, z: 0, w: 0),
    "inputGVector": CIVector(x: 0, y: s2, z: 0, w: 0),
    "inputBVector": CIVector(x: 0, y: 0, z: s2, w: 0),
    "inputBiasVector": CIVector(x: b2, y: b2, z: b2, w: 0)])
  .applyingFilter("CIColorClamp")
```

Now, put the two masks together.

You combine the masks by using the CIDarkenBlendMode filter, which chooses the lower of the two values of the input masks.
Then you scale the mask to match the image size.

```
let combinedMask = mask0.applyingFilter("CIDarkenBlendMode", parameters: ["inputBackgroundImage" : mask1])  let mask = combinedMask.applyingFilter("CIBicubicScaleTransform", parameters: ["inputScale": scale])
```

Finally, replace the return line with:
return mask

## Demo

You should see something like this [show the screen of the device]

Now the depth map is interactive, and you can change the thresholds to which the filter is applied. 

Next, you’re going to create a filter that somewhat mimics a spotlight. The “spotlight” will shine on objects at a chosen depth and fade to black from there.
And because you already put in the hard work reading in the depth data and creating the mask, it’s going to be super simple.
Open DepthImageFilters.swift and add the following:

Use the CIBlendWithMask filter and pass in the mask you created in the previous section. The filter essentially sets the alpha value of a pixel to the corresponding mask pixel value. So when the mask pixel value is 1.0, the image pixel is completely opaque and when the mask pixel value is 0.0, the image pixel is completely transparent. Since the UIView behind the UIImageView has a black color, black is what you see coming from behind the image.
You create a CGImage using the CIContext.
You then create a UIImage and return it.

```
func spotlightHighlight(image: CIImage, mask: CIImage, orientation: UIImageOrientation = .up) -> UIImage? {

  // 1
  let output = image.applyingFilter("CIBlendWithMask", parameters: ["inputMaskImage": mask])

  // 2
  guard let cgImage = context.createCGImage(output, from: output.extent) else {
    return nil
  }

  // 3
  return UIImage(cgImage: cgImage, scale: 1.0, orientation: orientation)
}
```

To see this filter in action, you first need to tell DepthImageViewController to call this method when appropriate.
Open DepthImageViewController.swift and go to updateImageView. Inside the .filtered case of the main switch statement, you’ll find a nested switch statement for the selectedFilter.
Replace the code for the .spotlight case to be:
finalImage = depthFilters?.spotlightHighlight(image: filterImage, mask: mask, orientation: orientation)
Congratulations! You’ve written your first depth inspired filter!

[TODO Faraz: Split here]

## Filter #2

Now let’s create another filter that highlights colour in a grayscale image. 

First we need to convert the image into a grayscale image. 

Open DepthImageFilters.swift and below the spotlightHighlight function

spotlightHighlight(image:mask:orientation:) 

you just wrote, add the following new method:

This should look familiar. It’s almost exactly the same as the spotlightHighlight(image:mask:orientation:) filter you just wrote. The one difference is that this time you set the background image to be a greyscale version of the original image.
This filter will show full color at the focal point based on the slider position and fade to grey from there.

The function will take in the image, mask, and current device orientation. 

We will have two filtered images; one greyscale image, and another output image which uses the greyscale image and blurs it with the original. 

Let's use the Photo Effect Mono filter to create the grayscale image. 

Now let's apply the blending mask between the original image and the greyscale image, while using our depth mask we created previously to focus on applying the colour to a specific depth in our new image. 

Now we just generate our image to a cgImage, and then convert it into a UIImage and return it. 

```
func colorHighlight(image: CIImage, mask: CIImage, orientation: UIImageOrientation = .up) -> UIImage? {

  let greyscale = image.applyingFilter("CIPhotoEffectMono")
  let output = image.applyingFilter("CIBlendWithMask", parameters: ["inputBackgroundImage" : greyscale,
                                                                    "inputMaskImage": mask])

  guard let cgImage = context.createCGImage(output, from: output.extent) else {
    return nil
  }

  return UIImage(cgImage: cgImage, scale: 1.0, orientation: orientation)
}
```

Open DepthImageViewController.swift and in the same switch statement for selectedFilter, replace the code for the .color case to actually use the new filter we just created:

```
finalImage = depthFilters?.colorHighlight(image: filterImage, mask: mask, orientation: orientation) 
```

Let's check out the filter we just created! You can use the slider to change which objects are in colour based on depth. 


## Filter #3

Don’t you hate it when you take a picture only to discover later that the camera focused on the wrong object? What if you could change the focus after the fact?
That’s exactly the depth-inspired filter you’ll be writing next!

Under your colorHightlight(image:mask:orientation:) method in DepthImageFilters.swift, let's add a blur function so that we can switch the focus of what we are looking at.

Let's name this function blur, which takes in an image, mask, and orientation. This method will return a UIImage. 

First, you create a mask that will invert the colour.
Then you apply the CIMaskedVariableBlur filter, which is new with iOS 11. This filter will blur using a radius equal to the inputRadius * mask pixel value. So when the mask pixel value is 1.0, the blur is at its max, which is why you needed to invert the mask first.
We don't want to blur what we are supposed to be focussing on! 

Once again, you generate a CGImage using the CIContext and use it to create a UIImage and return it.

```
func blur(image: CIImage, mask: CIImage, orientation: UIImageOrientation = .up) -> UIImage? {

  // 1
  let invertedMask = mask.applyingFilter("CIColorInvert")

  // 2
  let output = image.applyingFilter("CIMaskedVariableBlur", parameters: ["inputMask" : invertedMask,
                                                                         "inputRadius": 15.0])

  // 3
  guard let cgImage = context.createCGImage(output, from: output.extent) else {
    return nil
  }

  // 4
  return UIImage(cgImage: cgImage, scale: 1.0, orientation: orientation)
}
```


Before you can run, you need to once again update the selected filter switch statement. To use your shiny new method, change the code under the .blur case to use the new filter we created.

```
finalImage = depthFilters?.blur(image: filterImage, mask: mask, orientation: orientation) 
```

Now let’s run this and check out the filter. Notice how we can use a slider to focus on the object we want to focus on, after the fact!

## Talking Head

Alright, that’s everything I’d like to cover in this video.
At this point, you should understand how to deal with depth data, and how to create neat filters using depth.

I hope this screencast 
Anyways – I’m out!
