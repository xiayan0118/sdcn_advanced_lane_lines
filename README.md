# Advanced Lane Finding Project
---

## Goals
The goals / steps of this project are the following:

* Compute the camera calibration matrix and distortion coefficients given a set of chessboard images.
* Apply a distortion correction to raw images.
* Use color transforms, gradients, etc., to create a thresholded binary image.
* Apply a perspective transform to rectify binary image ("birds-eye view").
* Detect lane pixels and fit to find the lane boundary.
* Determine the curvature of the lane and vehicle position with respect to center.
* Warp the detected lane boundaries back onto the original image.
* Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.

Here I will describe each of these steps in my implementation. Code in this project can be found in [`P2.ipynb`](https://github.com/xiayan0118/sdcn_advanced_lane_lines/blob/master/P2.ipynb). All images and videos can be found in [`./output_images`](https://github.com/xiayan0118/sdcn_advanced_lane_lines/tree/master/output_images) and [`./output_videos`](https://github.com/xiayan0118/sdcn_advanced_lane_lines/tree/master/output_videos), respectively.

[//]: # (Image References)
[image1]: ./output_images/undist_chessboard.png "Undistorted Chessboard"
[image2]: ./output_images/undist_road.png "Road Transformed"
[image3]: ./output_images/color_gradient_thresholding.png "Binary Filter Example"
[image4]: ./output_images/warp_straight.png "Warp Straight"
[image5]: ./output_images/warp_curved.png "Warp Curved"
[image6]: ./output_images/lane_fits.png "Fit Visual"
[image7]: ./output_images/roc_pov.png "ROC and POV"
[image8]: ./output_images/all_tests.png "All test images"
[video1]: ./output_videos/project_video.mp4 "Video"

---
## Camera Calibration

The code for this step is contained in the code cell under heading **"Distortion Correction"** in [`P2.ipynb`](https://github.com/xiayan0118/sdcn_advanced_lane_lines/blob/master/P2.ipynb).

I start by preparing "object points", which will be the $(x, y, z)$ coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the $(x, y)$ plane at $z=0$, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the $(x, y)$ pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![alt text][image1]

## Pipeline (single images)

### 1. Distortion Correction for Lane Image

The function `distortion_correction()` under heading **"Distortion Correction for a Sample Road Image"** demonstrates this step. I use `test_images/test2.jpg` as a running example. Applying `cv2.undistort()` using the camera matrix `MTX` and the distortion coefficient `DIST` generated from previous step gives the result:

![alt text][image2]

### 2. Color and Gradient Thresholding

The function `gradient_color_threshold()` under heading **"Color and Gradient Thresholding"** demonstrates the steps for picking out lane line pixels in a grayscale image. This procedure gives the binary image below for the running example:
![alt text][image3]

* The 1st row: **Sobel filtering** on the horizontal dimension (x-axis) with min/max threshold `(35, 100)`. Performing this step on the x-axis is robust for detecting vertical lines, which the lane lines are most of the time.
* The 2nd row: **Gradient magnitude filtering** with min/max threshold `(80, 130)`.
* The 3rd row: **Direction filtering** with min/max threshold `(0.7, 1.3)`.
* The 4th row: **The combined gradient filter is the disjunction of the previous three (Sobel, Magnitude and Direction).**
* The 5th row: HLS color filtering on the S (Saturation) and H (Hue) channels.
* The 6th row: The final filter is the disjunction of gradient filter and the HSL filter

The HLS color thresholding is effective of picking out solid lines, while the gradient filtering is effective on dashed lines.

### 3. Perspective Transform

The code cells under heading **"Perspective Transform"** demonstrate the steps.

I first hard-coded the following source (`SRC`) and destination (`DST`) points by experimenting on a test image with straight lane lines (`'test_images/straight_lines1.jpg`):

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 200, 720      | 300, 720      | 
| 565, 470      | 300, 210      |
| 720, 470      | 950, 210      |
| 1125, 720     | 950, 720      |

Then I obtained the matrix for warp and unwarp images (`M` and `MINV`, respectively) using `cv2.getPerspectiveTransform()`. Then perspective transform is done using `cv2.warpPerspective()`.

I verified that my perspective transform was working as expected by drawing the `SRC` and `DST` points onto straight line image:

![alt text][image4]

The `SRC` points traces the trapezoidal region of lane lines well, and the warping using `DST` points gives parallel lane lines.

I also tried perspective transform on the running example, which has curve lane lines, and verified that the lines are indeed curved in the warped image:

![alt text][image5]

### 4. Lane Line Detection

The code cells under the heading **"Lane Line Dectection"** contains the steps. All states and methods for lane detection are encapsulated in the class `Line`. The function `detect_lanes()` uses the `Line` class and perform the following steps:

* Create two `Line` objects that represent the left and right lane lines.
* Split the image into left and right halves, and feed them to the corresponding `Line` objects.
* Each `Line` object `line.process()` the image by either `line.search_from_scratch()` or `line.search_around_poly()`, depending on whether a polynomial was detected in previous round. These methods identifies lane pixels, which will be used to fit a 2nd-order polynomial using `line.fit_poly()`. For a single image case, `line.search_from_scratch()` will always be used.
* Sanity check using `Line.sanity_check()`. This will compare the current polynomial fitting with the previous ones to make sure they are close enough. It will also compare the left and right fits to check:
	* They have similar curvature
	* They are separated by approximately the right distance horizontally
	* They are roughly parallel
* If the sanity check passes, we will do `line.success_update()` which will append the current fit to a running window of fits and update some counters.
* If the sanity check fails, we will increment a failure counter and redo fitting from scratch if the number of failures are above a threshold.

This procedure gives the following figure:
![alt text][image6]

The 1st row shows the visualization of lane detection from scratch and the 2nd row show that from a previously fitted polynomial.

### 5. Draw Lanes, Radius of Curvature and Position of Vehicle

The function `generate_output()` under heading **"Lane Drawing, Radius of Curvature and Position of Vehicle Marking"** demonstrates the steps.

* We mark the lanes using the fitted polynomials on the warped image.
* We then unwarp the image using `cv2.warpPerspective()` and the `MINV` matrix computed in step 3.
* Radius of Curvature is computed using `line.measure_curvature()` which implements the formula from lecture:
	* $$ R_{\text {curve}}=\frac{\left[1+\left(\frac{d x}{d y}\right)^{2}\right]^{3 / 2}}{\left|\frac{d^{2} x}{d y^{2}}\right|} = \frac{\left[1+\left(2Ax + B\right)^{2}\right]^{3 / 2}}{\left|2A\right|} $$
	* We compute the radius of curvature at the bottom of image ($x = 720$), and convert it from pixel to meters using a constant `YM_PER_PIX`. Comparing the radius of curvatures between left and right lanes, the one that's closer to 1KM is used for the image.
* Position of vehicle is computed by `line.compute_position()`.
	* For each line, we determine its bottom x coordinate ($x_l$ and $x_r$) using the best fit. The offset $o$ is computed as
	* $$ o = \frac{x_l + (w + x_r)}{2} - w = \frac{x_l + x_r - w}{2} $$
	* where $w$ is half of the width.
	* If $o > 0$, the vehicle is right of center, otherwise, it is left of center.
	* We use a constant `XM_PER_PIX` to convert this quantity from pixel to meters.

Below is the result of applying such procedure to the running example:
![alt text][image7]

### 6. Overall Pipeline

The function `process_image()` under the heading **"Overall Pipeline"** combining the steps above and converts a unprocessed image into a marked image. It:

* Distort correct the image
* Perform gradient and color thresholding
* Warp the image
* Detect lanes
* Draw lines, and mark radius of curvature and position of vehicle

I applied this pipeline to all test images and obtained decent results for all of them.
![alt text][image8]

---

## Pipeline (video)

Here's a [link to my video result](./project_video.mp4)

---

## Discussion

### Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

Problems:

* Many hyperparameter tuning steps. These hyperparameters are all tuned manually using 6 test images under similar conditions, so they will have problems generalizing to different environments. The bad performance of the current pipeline on challenge videos supports of this.
*  The challenge videos have different lighting/shadow and road conditions, and the hyperparameters and thresholds do not work well on them. The different road colors, shadows from trees and yellow on bright background are hard to be detected.
*  The current pipeline can produce unstable frames when the vehicle jumps or enter sharp turns.

Potential Improvements:

* More principled hyperparameter tuning. We can use blackbox optimization (such as Bayesian Optimization) to tune these hyperparameters jointl. For instance, we can collect representative videos and ask human to annotate the lane lines for these videos. Then we can formulate the hyperparameter tuning problem as a blackbox optimization problem as follows:
	* The blackbox function is $\mathbf{p} \rightarrow r$, where $\mathbf{p}$ is a vector of all hyperparameters in the pipeline, and $r$ is a scalar reward that we would like to maximize. For our example, $r$ can be the Intersection-over-Union (IoU) between the lane areas identified by our pipeline and the human labeled data.
	* Use Bayesian Optimization to optimize all hyperparameters that drive up the reward.
* We can of course also attempt other more Ad hoc improvements, such as try different color channels, use different thresholds for yellow and white lines and etc.