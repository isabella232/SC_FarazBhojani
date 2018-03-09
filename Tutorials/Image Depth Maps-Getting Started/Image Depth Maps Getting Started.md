# Screencast Metadata

## Image Depth Maps: Getting Started

## Create filters to alter photos using depth data. 



## Language, Editor and Platform versions used in this screencast:

* **Language:** Swift 4
* **Platform:** iOS
* **Editor**: XCode 9



## Talking Head

Hey what’s up everybody, this is Jessy. In this screencast, Faraz Bhojani are going to show you how to alter photos using depth data. By the end of the video we will have made a spotlight filter to change the position of lighting in an image based on depth. 

The iPhone 7+, 8 and 8+, and the iPhone X all have dual camera systems which allow you to capture depth information when taking pictures in portrait mode. If you've used a Microsoft Xbox Kinect, it's a similar idea. But with an iPhone, there's no infrared sensor involved.

[Slide 01]

This screencast is based on a tutorial created by raywenderlich.com team member Yono Mittlefehldt. If you like this screencast, be sure to give him a follow on Twitter.

And now, Faraz is going to implement some filters using depth data. 

## Demo
### 1_GettingStarted_initialDemo.mov

[Demo begins with starter project running on the screen]

I have a starter project here that shows some images, and you can tap to cycle between the images. It also has some buttons at the bottom to switch between different visual representations of depth data. They don’t do anything yet – that’s what we’re going to create together.

Let’s start with coding a visual representation for the depth in an image. We’re going to make it such that the closer an object is, the whiter it is, and the farther it is, the darker it will be.

You generally use AVDepthData to extract this auxiliary data from an image, so that’s the first step.

---

### 2_screencast_first.mov

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

---

### Missing?

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
 
[Slide 02]
 
In a nutshell, the iPhone’s dual cameras are imitating stereoscopic vision.

Try this. Hold your index finger closely in front of your nose and pointing upward. Close your left eye. Without moving your finger or head, simultaneously open your left eye and close your right eye.

The closer an object is to your eyes, the larger the change in its relative position compared to the background. Does this sound familiar? It’s a parallax effect!

[Slide 03]

Now quickly switch back and forth closing one eye and opening the other. Pay attention to the relative location of your finger to objects in the background. See how your finger seems to make large jumps left and right compared to objects further away?

[Slide 04]
 
The iPhone’s dual cameras are like its eyes, looking at two images taken at a slight offset from one another. It corresponds features in the two images and calculates how many pixels they have moved. This change in pixels is called disparity.
 
[Slide 05] 
 
The farther an object is, the smaller the disparity but the larger the depth. 
Depth is in fact the inverse of disparity and is equal to one over the disparity. 

You’re going to use this slider, along with the depth data, to make a mask for the image at a certain depth. Then you’ll use this mask to filter the original image and create some neat effects!

## Demo
### 3_screencast_masks

Open up DepthImageFilters.swift and find the createMask method with focus and scale. 

---

[Slide 06] 

The pixel value of your depth map image is equal to the normalized disparity. Remember, that a pixel value of 1.0 is white and a disparity value of 1.0 is the closest to the camera. On the other side of the scale, a pixel value of 0.0 is black and a disparity value of 0.0 is furthest from the camera.

We want to create a function that uses the depth in the image to convert the image to greyscale. The closer the pixel is, the whiter it will be, and the farther away it is, the darker it will be.

[Slide 07] 

However, if we simply converted the image this way as we did previously, it would be boring. So we're going to spice things up and create a filter function that focusses on a specific depth of the image so that we can use a slider to change which part of the image is in focus for the filter. The pixels around the focal point will be white and will ramp down to black around that depth. 

---
[Back to the code]

Before we can convert the depth data into an image mask, we need to define some constants. 

These help to create our filter function based on the slope of our mask parameters, filter width, and focus. 

[Slide 07] 

s1 is the slope of our function that defines how fast we ramp up from a black pixel to a white pixel towards the focal point. 

s2 is just the inverse of slope 1 for ramping back down after the focal point. 

The filter width is the entire length of the filter. Since we are only focussing on a particular focal point, this includes the ramp up, focal point, and ramp down. 

[Back to the code]

Note that to make the focal section a little more distinguishable, instead of only making a single point completely white, we make the pixels at that point white for a certain width (which is indicated by MaskParams.width). 

[Slide 08]

We will separate the filter function into two masks, ramping up (as you can see to the right)...

[Slide 09]

...and ramping down. All we did was break up that initial graph we saw before, into two parts.

[Back to the code]

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

---

## Demo

### 4_MaskDemo.mov

You should see something like this [show the screen of the device]

Now the depth map is interactive, and you can change the thresholds to which the filter is applied. 

Next, you’re going to create a filter that somewhat mimics a spotlight. The “spotlight” will shine on objects at a chosen depth and fade to black from there.
And because you already put in the hard work reading in the depth data and creating the mask, it’s going to be super simple.

---

### 5_spotlight_1.mov

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

---

### 6_spotlight2.mov 

Open DepthImageViewController.swift and go to updateImageView. Inside the .filtered case of the main switch statement, you’ll find a nested switch statement for the selectedFilter.
Replace the code for the .spotlight case to be:
finalImage = depthFilters?.spotlightHighlight(image: filterImage, mask: mask, orientation: orientation)
Congratulations! You’ve written your first depth inspired filter!

---

### 7_spotlightDemo.mov

Alright, that’s everything I’d like to cover in this video.
At this point, you should understand how to start working with depth data, and how to create a cool spotlight filter!

Speaking of… Apple’s so dumb… two cameras for depth??? I just use 3d glasses whenever I wanna see depth.
 
Anyways – I’m out!
