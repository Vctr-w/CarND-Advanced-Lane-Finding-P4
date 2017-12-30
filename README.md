# **Advanced Lane Finding**

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

[raw_calib]: ./images_for_report/raw_calibration.jpg "Raw calibration image"
[undistort_calib]: ./images_for_report/undistorted_calibration.png "Undistorted calibration image"
[raw_test]: ./images_for_report/raw_test.jpg "Raw test image"
[undistort_test]: ./images_for_report/undistorted_test.png "Undistorted test image"
[thresholded_test]: ./images_for_report/thresholded_test.png "Thresholded test image"
[untransformed_test]: ./images_for_report/untransformed_test.jpg "Untransformed test image"
[transformed_test]: ./images_for_report/transformed_test.png "Transformed test image"
[raw_test]: ./images_for_report/raw_test.jpg "Raw test image"
[windowed_test]: ./images_for_report/windowed_test.png "Windowed image"
[polynomial_test]: ./images_for_report/polynomial_test.png "Polynomial image"
[result]: ./images_for_report/final_image.png "Result"

## Camera Calibration

The second code cell of the IPython notebook "Advanced_Lane_lines_Notebook.ipynb" prepares the points used for camera calibration while the third code cell performs the calibration itself using the OpenCV function cv2.calibrateCamera().

First the object points (3d points in the real world space) are prepared. The object points (objpoints) are assumed to be on a flat image plane at z = 0 so that the object points are the same for each calibration image. For camera calibration, a chessboard of size 9 x 6 is used for the calibration. The OpenCV function cv2.findChessboardCorners() is called on each of the greyscale test images to find the image points of the 9 x 6 chessboard. These are appended to `imgpoints` while replicated object points are appended to `objpoints`. 

These are then used to compute the camera matrix and distortion coefficients which are used to undistort images using `cv2.undistort()`. An example of this undistortion is shown below.

![alt text][raw_calib]

![alt text][undistort_calib]

## Pipeline (images)

All processing of the image is done in the function `process_image()`.

The image is first undistorted in the same way the calibration images were undistorted.

![alt text][raw_test]

![alt text][undistort_test]

A combination of colour and gradient thresholds are used to generate a binary image. Lines 29 - 35 of the code cell containing `process_image()` perform this. 

Specifically a white pixel is one where the following conditions are met

```python
(Sobel_x & Sobel_y) | (S_thresh & V_thresh)
```

Sobel_x - Sobel operator in x orientation on a greyscale image where the values are thresholded between 5 and 255
Sobel_y - Sobel operator in x orientation on a greyscale image where the values are thresholded between 5 and 255
S_thresh - S in HLS image thresholded between 150 and 255
V_thresh - V in HSV image thresholded between 150 and 255

An example of this thresholding is shown below.

![alt text][thresholded_test]

A perspective transform is used to map the traffic lane to a bird's eye view from above. This allows us to calculate the curvature of the line. The parameters used to perform this transform are set in lines 4 - 19 which are used to perform the transform on line 41 using cv2.warpPerspective(). Reference points from a trapezoidal region were extracted from an undistorted image of a straight length of road. The trapezoid is projected onto a square with x side offsets so that sections just outside of the lane lines are visible. The source and destination points used are:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 598, 446      | 250, 0        | 
| 683, 446      | 1030, 0       |
| 1064, 681     | 1030, 720     |
| 241, 681      | 250, 720      |

To verify that the perspective transform has been performed correctly, a transform was performed on an image of a straight length of road and its warped counterpart was observed to have vertical parallel lines. 


![alt text][untransformed_test]

![alt text][transformed_test]

Using the warped thresholded image, lane line pixels were identified using a sliding window approach. The implementation of this is in the function find_window_centroids(). The algorithm splits the image into 9 horizontal strips and uses a window of width 50 for lane line search. It starts from two starting positions of the left and right lane (the centroid of the regions in the left and right half of the image containing the greatest number of thresholded points) and moves up each level, searching for the window within certain distance of the previous window that contains the greatest number of thresholded points. The function outputs these windows.

![alt text][raw_test]

![alt text][windowed_test]

Thresholded points are then extracted from these windows and a 2nd order polynomial is fitted. The implementation is on lines 49 - 87 of process_image().

![alt text][polynomial_test]

From these two polynomials, the radius of curvature is calculated. Assuming a lane length of 30m and width of 3.7m in the image, we can find the radius of curvature of the lines in metres. This is implemented on lines 104 - 116. The candidate points obtained previously are converted from pixels to m and then a 2nd order polynomial is fit onto these new points. Using the properties of a parabola, we then calculate the radius of curvature for both left and right lanes. The final result is the average of the two. 

To calculate the position of the car in the image, the camera is assumed to be mounted on the middle of the car. By finding the corresponding pixel in our warped image of the centre point of the image, we can calculate the offset from this point to the mid point between the two lane lines. Converting this to m gives us the position of the vehicle. This is implemented on lines 89 - 102.

Finally these polynomial is projected back to the undistorted image and displayed so that the results can be observed.

![alt text][result]

## Pipeline (images)

The final result video can be found [here](./output_project_video.mp4)

## Discussion

Some issues faced in my implementation were instability in calculating the radius of curvature. For lane lines where there were significant gaps between markers, sometimes the windowed search approach would fail if no thresholded pixels were found in a particular horizontal slice. For these cases, I kept the old window which works to some extent. 

Due to the nature of the perspective transform, pixels further away have a greater contribution in the warped image and so inaccuracies in thresholding further away result in issues in detection the lane lines accurately. More fine-tuning of the thresholding could make this more robust. 

This implementation could possibly fail if there is a car in front of our camera. In this case, we may not be able to accurately find the base of the line if much of the car creates thresholded pixels in our thresholded image. This is likely to happen since cars tend to be full of edges and quite vibrant in colour. For this particular case, it is likely we need to occlude middle sections of the image from the search. This will fail if there is a car in our frame crossing a lane line. For this particular case, we may need to have some reliance on historical measurements. 
