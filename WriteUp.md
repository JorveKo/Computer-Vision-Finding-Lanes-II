## Writeup Template

### You can use this file as a template for your writeup if you want to submit it as a markdown file, but feel free to use some other method and submit a pdf if you prefer.

---

**Advanced Lane Finding Project**

The goals / steps of this project are the following:

* Compute the camera calibration matrix and distortion coefficients given a set of chessboard images.
* Apply a distortion correction to raw images.
* Use color transforms, gradients, etc., to create a thresholded binary image.
* Apply a perspective transform to rectify binary image ("birds-eye view").
* Detect lane pixels and fit to find the lane boundary.
* Determine the curvature of the lane and vehicle position with respect to center.
* Warp the detected lane boundaries back onto the original image.
* Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.

[//]: # (Image References)

[image1]: ./RubricRequirements/undistortedChess.png
[image2]: ./RubricRequirements/undistortedTestPic.png
[image3]: ./RubricRequirements/BinaryImage.png
[image4]: ./RubricRequirements/WarpedBinaryImage.png
[image5]: ./RubricRequirements/FinalImage.png
[image6]: ./RubricRequirements/FinalImage.png
[video1]: ./RubricRequirements/FinaleVideoPipeLine.mp4

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup 

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

Within the function "camCalib()" I calculate the Matrix as well as the coefficients needed for the correction of the image.
To do that I grab the objects points which are easily defined by the inner corners. The imagepoints are looked up by looking at at a grayscale picture of the chessboard image and then use the "findChessboardCorners" function.
I looped through all of the provided chessboard images to get the required output.
The function "undistortIMG()" then uses theses values to undistort any image given with this camera.
See the image below (or in this folder undistortedChess.png) for the undistorted example.

![alt text][image1]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

If I apply my correction to an actual image of the road it looks like this (or in this folder undistortedTestPic.png)
![alt text][image2]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

The function "createBIN()" in my pipeline is handling the transformation to a binary image.
At first I transform the picture to the HLS space to be able to use these values for further processing.
To do that i split up the image in the different channels of the HLS space.
For the lanes are most likely to be vertical I take the x derivative with the sobel() operation which also includes a gaussian blur fitler, hence I don't need to manually do it beforehand to reduce noise.
I then create the first threshold with the x gradient in sobel. The coordinates that fullfill the threshold receive a "1" the rest are left at "0".
I then also do a saturation and hue threshold in the same manner to then combine of all of my binary results into one.
So if either of them is "1" at any coordinate the final result is also "1" at that spot. 
Look at the image below or in the folder to see the example output (BinaryImage.png).

![alt text][image3]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

I first gathered the required parameters for each warp with the function "warpParameters()".
In that I took an undistorted image to then fit int a polygon (trapez) similar to masking in the previous project that fit right in the lanes and said that these are the source points and for the destination points I just said that this should be a rectangle for the lanes were straigh in the chosen frame.
I acutally manually grabbed the points for this and had multiple loops to find the best result.
This could probably be improved by aumating this task but for now I was satisfied with the results.
I then calculated the M matrix and its inverse to be able to warp and unwarp any frame with the functions "warpIMG()" and "unwarpIMG()".
I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.
So it was working pretty well. Below you can see an example or in this folder (WarpedBinaryImage.png)

![alt text][image4]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

To find the lanes on the warped binary images I used "findFirstLanes()" for the first image. Hence I am not going in too deep on this function. It takes the histogram to find the starting point on the bottom of the picture of the lane.
With that it builds a number (e.g. 9) boxes to the top of the image. These boxes have a width (e.g. 100 px) and height (set by number of boxes). THe pixels within each box are counted and if a certain number of "good" pixels is in the box the next box is readjusted to the mean value in the the horizontal axis. Hence the boxes build up along the lane. Then all "good" pixels which are within all of the boxes are used to fit a polynomial of the second order.
This happens in "fitPoly()" which acutally calls the "findFirstLanes()" from within itself. And then returns the points of the polynomial to then draw on the image.

For each frame after the first I used the "fitPolyPP()" function. In this function I tried a lot of stuff with history saving of previous polys to improbe the jittering but then decided to do this in the function "search_around_poly()" which is acutally calling the "fitPolyPP()" from within itself. so the "fitPolyPP()" is pretty much just liek the regular "fitPoly()" funcion.
I just left it in to keep my thoughts in the document even if it does not look very "clean" right now.
So in this function "search_around_poly()" I use the polynomial from the previous frame to check left and right from it within a certain margin for the next "good" pixels that should contribute to our new polynomial.
I set the margin to 40 to look in both x directions from the poly right now.
In here I then I am creating a more stable solution by saving the lower "good" pixels of earlier images left and right to a history. For the last 11 images. I then add these points to each new processed image create a more steady output.
Within this history I also reject more outliers by making sure that values that are too far off the average of the last frames in that region are also rejected.
This gives me pretty solid output.
I then take the points once they have been fitted and draw the lanes as well as a green area inbetween the lanes on the image.
You can see the result below or in this folder (FinalImage.png)


![alt text][image5]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

At the end of the "search_around_poly()" function I am calculating the distance to the center of the lane.
I am checking how many pixels are between the two lanes at the bottom of the picture and then take the center of the picture taken and then subtract them to see where the middle of the camera is (=center of car) in regards to the middle of the lanes.
With the transformation from pixels to m I get the result in real world measurements. It seems to be always around 1.5m off. I think this is becaus the camera seems to be mounted to the left of the car center and not in the acutal center of the car.
The radius is calculated within my final pipeline function "pipeLineConti()" at the end.
I ahve taken the derived functions from the previous exercise and fed my data into it.
I then at the end print the curvature and distance on the screen

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

Here you can see the final result image.


![alt text][image6]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./project_video.mp4)
Can also be found in this folder (FinaleVideoPipeLine.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

I spent hours on tweeking my parameters for the binary image as well as for the margins on the poly fitting. I found that having a good preprocessing of the image takes so much load off the later steps. 
Sometimes the end points of my lanes can jump off for 2-3 frames to then get back to the lane if the car rides over a bump for example. I tried also implementing the same history as for the bottomg parts of the lanes. But that did not work at all and I couldn't make it work. Becaus if I did that and the lane cam back from a left curve to a straight lane for example, the history caused the right lane to just wander of into the left lane because it still had the left curve in its history. I tried multiple ways to fix this but I did not find anything to do so. Even if I just kept 1 previous frame in the history it happened. Probably because at the end of the lane there aren't many pixels to hold up against the history and to get the lane back to where it belongs.
For the challenge videos the result is pretty bad with this. But I think I need to put even more effort into preprocessing. To get maybe just the yellow and white color channels etc. 
I also think that it could be worht to adapt the binary image process by make the pipeline recognize the lighting situations and background color of the road and then adapt the thresholds for each scene individually.

I would like to have implemented a more robust way for the end points of my lanes. If possible I would love to hear feedback on that.

Sorry for having a lot of boilerplate code in my notebook. But when I later go back to my project, I want my old code still in there to get back to some of my thought processes. Even if it makes it less readable.


Thank you,
Jorve


