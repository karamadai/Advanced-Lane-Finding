

***Advanced Lane Finding Project***
-----------------------------------

The goals / steps of this project are the following:

* Compute the camera calibration matrix and distortion coefficients given a set of chessboard images.
* Apply a distortion correction to raw images.
* Use color transforms, gradients, etc., to create a thresholded binary image.
* Apply a perspective transform to rectify binary image ("birds-eye view").
* Detect lane pixels and fit to find the lane boundary.
* Determine the curvature of the lane and vehicle position with respect to center.
* Warp the detected lane boundaries back onto the original image.
* Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points
Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---
### Code Structure
The code is organized into the following classes:
1. **Camera class**: This class exposes methods that perform camera calibration and undistord the image (line 14-81)
2. **ImageProcessor class**: This class exposes a method that applies gradient, gradient magnitude, hls color threshold to return a thresholded binary image (line 87-156)
3. **Lane class**: This class is used to store the polynomial fit and other lane information. This is used to average the fit over a specified number of frames (12) to smooth the rendering of the lanes (line 161-189)
4. **LaneFinder class**: This class exposes methods that identity the lanes on the binary image. The lanes are detected using the histogram with sliding window and extrapolation methods (line 195-414)
5. **DrawLanes**: This class exposes methods that instantiates and invokes appropriate methods to create binary image, change perspective, render the lanes on the image , compute the lane curvature and vehicle position (line 417-495).
6. **Main Method**: This is the entry point to the program.  The camera calibration, reading the video file, instantiating the DrawLanes class to render lanes and saving the output video is done here (line 499)

###Camera Calibration

The camera calibration is performed by the Camera class in the calibrate_camera method().  The calibration is done using the 9x6 chessboard calibration images provided by Udacity.  The output of the calibration is pickled so that the calibration is run only once to save time.  

The undistort() method reads the calibration matrix and the distortion matrix from the pickle file to undistort a given image.  The un-distortion is performed using the cv2.undistort() funtion. 

The image below shows the distorted and un-distorted calibration images
![enter image description here](https://raw.githubusercontent.com/neelks72/AdvancedLaneFinding/master/calibration_image.png)

###Pipeline (single images)

####1. Example of a distortion-corrected image.

Once the camaera is calibrated the cv2.undistort() method is used to correct for distortions. The following image show the output of undistort method when applied to a test image

![enter image description here](https://github.com/neelks72/AdvancedLaneFinding/blob/master/Distored_Undistored.png?raw=true)

####2. Binary Image Creation

The ImageProcessor Class exposes methods to create the binary image. The binary image is created at line 452. The following set of thresholds are applied to the image to obtain the final binary image:

1. **RGB Color Threshold**: A color threshold of maximum 250 and minimum 220 is applied only to the Red channel. The following image shows the Red channel thresholded image:
![enter image description here](https://github.com/neelks72/AdvancedLaneFinding/blob/master/red_threshold.png?raw=true)

2. **HLS Color Threshold**: The image is converted into the HLS space and a max, min threshold of 255,180 is applied only the S Channel. The image below shows the S channel thresholded image
![enter image description here](https://github.com/neelks72/AdvancedLaneFinding/blob/master/mag_threshold.png?raw=true)

3. **Gradient Thresholds**:  The gradient threshold is applied with the gradient maximum and minimum of 100, 20 and Sobel kernel size of 5. The following image shows the out of the gradient threshold
![enter image description here](https://github.com/neelks72/AdvancedLaneFinding/blob/master/grad_threshold.png?raw=true)

4. **Combined Binary Image**: The final binary image is obtained by combining the all the above thresholded images (line 133) . The image below shows the final thresholded binary image:
![enter image description here](https://github.com/neelks72/AdvancedLaneFinding/blob/master/final_threshold.png?raw=true)


####3. Prespective Transformaion.

The perspective transformation is preformed by the __change_prespective method of the LaneFinder class (line 209).  The following hard coded source and destination points where used. These points were arrived at by trail and error:

 src = np.float32([[280, 665], [1035, 665], [520, 500], [780, 500]])
 dst = np.float32([[205, 665], [1075, 665], [200, 470], [1105, 470]])

The validity of the perspective transform was validated by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image as shown below:
![enter image description here](https://github.com/neelks72/AdvancedLaneFinding/blob/master/Binary_Warped.png?raw=true)


####4. Lane Line Identification and Polynomial Fit

The LaneFinder class contains the methods for identifying the lane line pixels in the binary warped image. The code uses histogram to identify the base points of the left lane and right lane pixels (lines 268). Once the base lane pixels is identified, the code uses the sliding window technique to identify the rest of the lane pixels. Once the lane lines pixels are identified the second order polynomial fit is obtained using the cv2.polyfit() method (line 306-307) for the left lane and right lanes. The following image shows the output using the sliding window method.
![Sliding Window Output](https://github.com/neelks72/AdvancedLaneFinding/blob/master/sliding_window_lane.png?raw=true)

For the subsequent frames, the lane line pixels are identified using the extrapolation method. I used the right and left lane polynomial fit identified in the previous frame and a  search window (margin)  100 pixels wide.  Once the lane lines pixels are identified the second order polynomial fit is obtained using the cv2.polyfit() method (line 346-347) for the left lane and right lanes. The following image shows the lane lines pixels 
![Extrapolated lanes](https://github.com/neelks72/AdvancedLaneFinding/blob/master/extrapolated_lane.png?raw=true)

####4. Sanity Check
To ensure that the lanes lines identified above are valid I perform the following checks (lines 226-259):
1, Horizontal distance check: Ensure that the difference between the max and min space between left and right lane pixels do not exceed lines more than 10% of the mean distance between the lanes.
2. Parallel lanes check: Ensure that left and right are parallel to each other. This done by comparing the slope angle of the secant line going the start and end pixels of the left and right lanes does not deviate more than 20 degrees 

If the sanity check fails for a given frame, the frame is re-evaluated using the sliding window method.  If the output of the sliding window method also fails the sanity test, the polynomial fit for that frame is ignored and the lane lines are drawn using the buffered values from the previous valid frame. (line 457)

####5. Radius of Curvature and Vehicle Position

The radius of curvature is calculated using the 12 frame averaged polynomial fit (line 436).  The code assumes the lane is 3.7 meters wide to compute the distance in meters each pixel represents.  The vehicle position is determined in lines (483-488)

####6. Final Image with Lanes Drawn 

The frame averaged (12 frames) polynomial lane is first drawn on warped blank image using cv2.polyfill() method (lienes 463-479) . This image is then wrapped back to the original perspective using the inverse perspective matrix obtained during the creation of the binary warped image. The following image shows the final result:

![enter image description here](https://github.com/neelks72/AdvancedLaneFinding/blob/master/lane_rendered.png?raw=true)
---

###Pipeline (video)

The pipe line video is included in the zip file.  

Here is the link to my youtube upload (click image to view in youtube):

[![IMAGE ALT TEXT](https://github.com/neelks72/AdvancedLaneFinding/blob/master/youtube.PNG?raw=true)](https://youtu.be/y5hOG8MIOgM "Advanced Lane Finding")
---

###Discussion

####1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

The pipe line is likely to fail in merging lane conditions and  possibly at certain intersections with criss-crossing lane lines.  Other road markings such as straight "ahead arrows" or "turn arrows" painted on the road might create noise that will need to be filtered out.  The code can be made robust by enhancing the Sanity check functions to include validations to measure the distance between lane marking along the y-axis along with curvature validation.
It will be interesting to see how the code performs in real time.  The code has  room for optimization to improve performance.   
