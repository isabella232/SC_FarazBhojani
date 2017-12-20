# Screencast Metadata

## Image Depth Maps: Color Highlighting and Focal Length Filters

## Create filters to alter photos using depth data. 



## Language, Editor and Platform versions used in this screencast:

* **Language:** Swift 4
* **Platform:** iOS
* **Editor**: XCode 9


## Talking Head

Hey what’s up everybody, this is Faraz. In this video, I’m going to show you how you can create filters to alter photos using depth data. Specifically we will create a filter to highlight colour in an image, and another filter to change the focus within an image. 

The iPhone 7+, 8 and 8+, and the iPhone X have a dual camera system which allows  you to capture depth information when taking pictures in portrait mode. 

“This screencast is based on a tutorial created by team member Yono Mittlefehldt
 If you like this screencast, be sure to give him a follow on twitter.”

Alright, now lets implement some filters using depth data. 

## Demo

[Demo begins with starter project running on the screen]

I have a starter project here that shows a depth mask and a spotlight filter. It also has some buttons at the bottom to switch between different visual representations of depth data. A couple don’t do anything yet – that’s what we’re going to create together.

## Highlight Colour Filter

Let’s start by creating a filter that highlights colour in a grayscale image. 

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

Alright, that’s everything I’d like to cover in this video.
At this point, you should understand how to deal with depth data, and how to create neat filters using depth.

Speaking of... if your phone can't change its focus, it may need a new prescription. 

Anyways – I’m out!
