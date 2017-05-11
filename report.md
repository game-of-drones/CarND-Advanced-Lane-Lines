##Advanced Lane Project Lines Report

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

[image1]: ./report/undistort.png "Undistorted"
[image2]: ./report/test1_undistort.png "Road Transformed"
[image3]: ./report/binary_combo_example.png "Binary Example"
[image4]: ./report/warped_straight_lines.png "Warp Example"
[image5]: ./examples/color_fit_lines.jpg "Fit Visual"
[image6]: ./report/final_output.png "Output"
[video1]: ./examples/output_project_video.mp4 "Video"
[docs]:   ./report/warpPerspectiveDocs.png "docs"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points
### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---
### Writeup / README

### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!

I tried to use the ipython notebook to document the whole project in detail.

### Camera Calibration

### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the first section of the IPython notebook located in "./examples/example.ipynb". I wrote 2 helper functions:
`prepare_calibration` and `calibrate_camera`.

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

Actually, I failed to detect all corners in calibration1.jpg, calibration4.jpg and calibration5.jpg. But the rest of the calibration images which I successfully detected the corners are good enough to generate good calibration results.

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![alt text][image1]

### Pipeline (single images)

### 1. Provide an example of a distortion-corrected image.
After doing the calibration and obtained the camera matrix and distortion coefficients, I use `cv2.undistort` to apply the distortion correction to one of the test images like this one:
![alt text][image2]


### 2. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

Instead of doing color and gradient thresholding to the undistorted image, the second step of my pipeline is perspective transformation. Here the goal is to obtain the `M` and `M_inv` matrices using `cv2.getPerspectiveTransform`. The code is located in the 2nd section of my ipython notebook, and the function is called `perspective_transform`.

The key of `perspective_transform` is to select the 4 points from the source image that you know form a rectangle, and 4 points in the desired bird-eye view image which actually form a square. I use "test_images/straight_lines2.jpg" as my input image, and selected my source points and destination points as follows:

```
# top left, top right, bottom right, bottom left
# Hand picked points that form a rectangle in original image
src = np.float32([[555, 476], 
                  [733, 476], 
                  [1035, 665], 
                  [286, 665]])

dst = np.float32([[200, 10], 
                  [1080, 10], 
                  [1080, 710], 
                  [200, 710]])

```

One thing to note is that I intentionally make the lane take most of the image and exclude the curbs from the resulting image. This makes the next steps, like color/gradient thresholding and lane detection much easier. Otherwise, the shadow from the lane divider can sometimes cause confusion for the lane detector. 

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][image4]


After getting the calibration parameters and the perspective transform matrices, I use pickle to save them in a file called "saved_params.p", so I don't have to rerun these steps every time. This part of the code is located at the 4th cell of the ipython notebook. 

### 3. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

This part is contained in section 3 of my ipython notebook.

I used a combination of color and gradient thresholds to generate a binary image.  Here's an example of my output for this step.  (note: this is not actually from one of the test images)

![alt text][image3]

### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

Detect lane line pixels consists of the following steps:

1. Use histogram at the bottom 2/3 of the binary image. The peaks at the left and right part of the histogram are good guess of the overall x coordinates of the lanes. I call them leftx_base and righx_base.

2. Along the lane line, I divide the each lane into 9 sections. For each section, I search non zero pixels in a window. Taking the left lane as an example, at the bottom, the center of the window is leftx_base. Inside this window I find all the non zero pixels, adding them as pixels of the lane, and calculate the average of their x coordinates. The I use this average x coordinate as the center for the window of the next section.

3. After finishing searching all points of the each lane, I fit these points with a 2nd order polynomial

The above procedure, which is implemented in function `find_lane_sliding_window` in section 4 of my ipythohn notebook, assumes there is no previous knowledge of where the lanes are. However, if I already have a good fit from previous video frame, I can simply search points within a certain distance from my previously identified and fitted lane. This idea is implemented in function called `find_lanes_without_window`.

### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

We already have the equations to calculate the radius of curvature for our fitted polynomials. However, we need to convert the unit from pixel to meters. To determin the parameters `x_meter_per_pixel` and `y_meter_per_pixel`, I measured in the warped image and found that the width of the lane (3.7 meters) taks 854 pixels, and the gap in the dashed linne (3 meters) takes 170 pixels. So we have 
```
  x_meter_per_pixel = 3.7/854
  y_meter_per_pixel = 3.0/170 
```
The code for calculating the radius of curvature is in function `measure_curvature`.

To find the position of the vehicle with respect to the center, we need to identify the center of the lane in the ***original camera view*** (instead of the warped image, or the bird-eye view),  compare it with the center of the lane (1280/2), and then convert the difference into meter.

The problem is, we only identify the lanes in the warped image, and how to convert the x coordinates of the lanes in the warped image to those in the original image. Searching documentation of warpPerspective, I found how to use the `M` and `Minv` matrices to convert a point's coordinate from a source image to the coordinate in the destination image.

![alt text][docs]

In our case, we know the coordinate of the bottom point of a lane in the warped image (xwarp, ywarp), we want to find the corresponding point in the original undistorted image (xundist, yundist). From the formula above, we select destination image to be the warped image, and the source image to be the undistorted image, and our matrix is `M`.

The implementation of this step is in function `find_distance()`.


### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step in function `draw_result_in_car_view()`.  Here is an example of my result on a test image:

![alt text][image6]

### 7. Full pipeline

My full pipeline is in function `my_pipeline()`.
---

### Pipeline (video)

### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./examples/output_project_video.mp4)

---

### Discussion

### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

This project is very challenging. 

I found that the color and gradient thresholding is very hard to tune. You have different channels in different color spaces, and each of them will behave differently under different condition. I chose to do gradient threshold on grayscale image and color thresholding on the S channel in the hls color space. However, I could do gradient threshold also on S or even L channel. For each combination I chose, there are threshold max and min to tune. Without a good binary image, it's hard to continue with the lane detection and polynomial fitting.

I tested my pipeline on the challenge video, and found that it detect the uneven color of the middle of the lane as lane line. 

In my code, the `find_lanes_without_window` function find two lanes. I think I should treat each lane seperately so that we can decide if the detection is good for each lane and if we can skip the moving window, like the line class suggested by section 36 of the course.
