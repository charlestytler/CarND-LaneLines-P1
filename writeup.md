# **Finding Lane Lines on the Road** 

---

**Finding Lane Lines on the Road**

The goals / steps of this project are the following:
* Make a pipeline that finds lane lines on the road
* Reflect on your work in a written report


[//]: # (Image References)

[originalImage]: ./test_images/solidYellowLeft.jpg "Original"
[cannyImage]: ./test_images_output/solidYellowLeft_cannyEdge.jpg "CannyEdge"
[groupsImage]: ./test_images_output/solidYellowLeft_groups.jpg "Groups"
[lanesImage]: ./test_images_output/solidYellowLeft.jpg "Lanes"

---

### Reflection

### 1. Describe your pipeline. As part of the description, explain how you modified the draw_lines() function.

My pipeline can be described in 3 stages: line identification, line downselect, and lane estimation.

#### Line Identification
The first stage is taking advantage of the OpenCV toolset to identify lines in the image. The image is first converted to grayscale and a Gaussian blur is applied to reduce noisy gradients in the image. Then Canny edge detection is applied to the blurred image to isolate all edges in the image. Since the camera perspective is fixed with respect to the vehicle traveling in the lane, we can take advantage of known geometry of the image and mask areas we know lanes won’t exist. We know that the lane lines will be on the ground, so the upper portion of the frame can be ignored (I masked off the upper 60% of the image). We also know that due to perspective, the lanes will angle closer together with distance, so the left and right sides of the image are masked at an angle to help ignore neighboring lanes. Finally, a Hough transform is used to identify straight lines of significant length from the collection of edges in the unmasked region.

#### Line Downselect
Now that a collection of lines have been identified, they need to be down selected into which ones are representative of the left and right lane markings. So first, the set of all lines are grouped into subsets of lines with similar angles in the image. To allow for possible curvature in the lanes, the lines are compared from bottom of image to top, using the top most line as the reference angle. Any lines within +/-10 degrees of the reference angle are added to the group, lines which are near horizontal (by +/-20 degrees) are removed altogether. After a set of line groupings have been sorted, the two groups with the greatest cumulative line lengths are retained as the left and right lane line groups. This is based on the assumption that within the masked region, the lane markings should be the most prominent lines detected by the Hough transform.

#### Lane Estimation
After the lines have been separated into left and right lane groupings, an estimate of the full lane boundary can be estimated. For a series of y-axis coordinates (3) ranging from the bottom of the image to the highest y-coordinate which both left and right line groupings reach, an x-coordinate is determined for each lane group. This x-coordinate is the average of the x-values for which each of the nearby lines would pass through if projected to the given y-coordinate. The x-coordinate for the bottom edge of the image is determined by extrapolating from the 2nd from bottom point using the average angle of the lines in the group. This allows some bend in the lane projection for curving lanes.

Finally, to avoid high frequency effects from noisy line identification a low-pass filter was applied to the lane estimates. The x-coordinates and y-coordinates were independently filtered based on their previous value. New measurements which exceeded a pixel distance threshold from the past estimate were rejected outright, and those within the threshold were only gradually incorporated into the lane estimate by only applying a fraction of their provided innovation from the previous estimate. This allowed steady incorporation of changes in the lane as the car travels, but avoids jumpy behavior and ignores outlier lane line measurements.


Original Image:

![Original image][originalImage]

Image after Canny edge detection:

![Image after Canny edge detection][cannyImage]

Image with Hough transform detected lines after being organized into groups of similar angle (groups are color coded). The yellow and green groups will be retained as the two with the greatest cumulative line lengths.

![Image with Hough transform detected lines color coded by groups][groupsImage]

Final image with superimposed lane estimates:

![Final image with lane estimates][lanesImage]



### 2. Identify potential shortcomings with your current pipeline


Shortcomings:
Right now my filtering of lanes rejects new measurements if they are too far from past estimates. If the image loses tracking long enough and the actual lanes drift away from past estimates, the estimator won’t be able to recover until the actual lanes come within its threshold again.

Also, the line detection relies on a good contrast between pavement and the lane markings. In situations with poor lighting, dirty roads, or lighter colored pavement it has trouble finding any lines. An example of this is seen in the Challenge video where the car drives over a patch of light-gray pavement and the line detection loses track of the lanes and relies on the past estimates until it regains a valid measurement. Improved performance could possibly be achieved through further adjustments of the Canny edge detection parameters and its preceding steps.


### 3. Suggest possible improvements to your pipeline

Some potential means for improving the accuracy of the lane estimates could be:
* Color filtering -- I did not take advantage of the lane colors which are known to be yellow or white. I did this because I did not want color filtering to add noise that the Canny edge detection would pick up. However, isolating white and yellow colored pixels could aid in lane marking identification.
* Smart masking -- One could use known camera angles and/or previous lane estimate and a prediction of how the lanes could possibly change or curve further down the road to apply tighter masking.
* Using independent sensors or information to aid lane prediction. For example, a known steering angle or inertial sensor could give knowledge of future camera perspective, and thus a prediction of future lane position in the image.
* Use knowledge of pavement marking regulations to create stricter rules for lane line identification, such as minimum and maximum dashed lane line gap lengths.
* Lane angle -- Right now lines close to horizontal are ignored, but that is the only angle restriction. Stricter angles could be used for line downselection based on knowledge that the car/camera is between the two lane lines or on past lane estimates.
* Filter improvements -- The measurement rejection logic of my filter could be improved so that it resets if it gets stuck in a state where it is consistently rejecting all new measurements.
* Curve fitting -- Instead of using straight line segments for the lane boundary estimate, a more accurate representation could be found if a higher order curve were fitted to the measurements.
