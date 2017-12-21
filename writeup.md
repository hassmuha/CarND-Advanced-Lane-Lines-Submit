
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

[image1]: ./test_images/original_image.jpg "Checkerboard Original Image"
[image2]: ./examples/undistort_image.jpg "Undistorted Checkerboard"
[image3]: ./examples/test1.jpg "test Image"
[image4]: ./examples/test1_undistorted.jpg "Test Image undistorted"
[image5]: ./examples/gradx_L.jpg "L-plane grad-x binary"
[image6]: ./examples/s_binary.jpg "S-Plane threshold binary "
[image7]: ./examples/h_binary.jpg "H-Plane binary "
[image8]: ./examples/combined_sh.jpg "Combined S & H planes binary"
[image9]: ./examples/combined_binary.jpg "Final Binary"
[image10]: ./examples/roi_edges.jpg "Applying Region of Interest"
[image11]: ./examples/warped.jpg "Wraped Image"
[image12]: ./examples/sliding_window.jpg "Sliding Window Search and Polynomial Fit"
[image13]: ./examples/Final_result.jpg "Final Result"
[video1]: ./project_video_out.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the first code cell of the IPython notebook located in "./examples/example.ipynb" (or in lines # through # of the file called `some_file.py`).  

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result:

![alt text][image1]
Undistorted Image
![alt text][image2]

### Pipeline (single images)

let's start with test1 image as shown below
![alt text][image3]

#### 1. Provide an example of a distortion-corrected image.

I have defined a new funciton so that same function can be used for processing video frames as well. This is defined as `distort_correct()` which is using the opencv function `cv2.undistort()` and calibration and distortion cooefficients calculated as describe above to calculate the undistorted test image. Both the original and undistorted image is shown below:
![alt text][image4]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I used a combination of color and gradient thresholds to generate a binary image.  I used following steps as present in the function `pipeline()` :

1) Color space conversion : RGB to in HLS
2) Applying guassian smoothing on L plane to prepare to calculate gradient on that plane. The video seems alread blurred so I used the mask of 3 x 3.
3) Apply x-gradient with Sobel mask of size of 5 on L-plane. The L plane is selected as it contain major features of the whole scene along with presence of clear edges. Based on the fact that we are interested on lane lines x gradient was enough. The threshhold image `gradx_L` is calculated by applying thresholding on the absolute of gradient image with values between 20 to 100 are considered as white. This is shown in the figure below:
![alt text][image5]

4) Simple thresholding is applied on S-plane with values between 140 to 255 taken as white. The reason is it will help to detect yellow and clear white lines if present in the image while not being effected by different change in lighting conditions. The resulted image is store in the variable `s_binary` and picture is shown below
![alt text][image6]

5) Simple thresholding is applied on H-plane with values between (15, 40) taken as white. The result shown below also shows it will also be able to detect yellow and white lines. The resulted image is store in the variable `h_binary`:
![alt text][image7]

6) Then combine `s_binary` and `h_binary` with 'and' operation. this reduces a lot of noise and make sure pure lane lines will be detected. The resulted image is store in the variable `combined_sh` and resulted image is shown below:
![alt text][image8]


7) Then combine `combined_sh` with `gradx_L` with 'or' operation. Mostly lane lines near to the camera were clearly identified by the H and S plane thresholded images, while the further far lane line with curvature information is present in `gradx_L` image. That is the reason of using or operation. Here's an example of my output for this step:

![alt text][image9]

7) At the end Region of Interest based thresholding technique is also applied to remove much background and left with road views

![alt text][image10]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

- performed on done on the straight line straight_lines1.jpg
For taking the perspective transformation I selected four points on the road and they are more or less form a shape of trapezoid as shown in the picture below and mapped it to a rectangle. The code is show in the line ''.

This resulted in the following source trapezoid and destination rectangle points are given in the table:

| Source        | Destination   |
|:-------------:|:-------------:|
| 585, 460      | 320, 0        |
| 203, 720      | 320, 720      |
| 1127, 720     | 960, 720      |
| 695, 460      | 960, 0        |

Moreover  `cv2.getPerspectiveTransform(src,dst)` function is used to get the perspective transformation and defined the my own `wrap_perspectTransform()` which takes the perpective matrix and the image as input and using `cv2.warpPerspective()` transformed the input image and returns the warped image as output. For showing the results I am taking the undistorted thresholded image as input and after transformation wraped image is shown below:

The lane lines appear parallel in the warped image, this provide us confidence that the transformation is working fine and later using the angle of curvature we can also confirm this.

![alt text][image11]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

I have used the two functions as provided in the lecture `slidingwindow_polyfit(...)` and `lookAhead_polyfit(...)`
this uses the technique of partitioning the whole image into segments along y axis. Then find possible location of left and right plane starting from the bottom segment and moving up till the first segment.
For the first segment histogram is calculated along x-axis. Then pass a separate window search to find peaks of the histogram which are parts of left and right lane. The final position of window in each segment also serves as starting point to find the left and right lane in next upper segment, so called seed. Widow is move left and right to find best peak. And keep applying these steps till the top segment of the image. For each segment and for each lane I took center of the window points to fit 2nd order polynomial function. This is done in by using `numpy.polyfit(...)` function.

In the figure below the detected widow peak is shown along with curve line is also drawn which is calculated based on the polynomial coefficients found :

![alt text][image12]

But for the video pipeline `lookAhead_polyfit(...)` is also used, this step will be explained later but main idea here is not to perform the sliding window step to avoid computation and exploit the temporal locality fact that in two consecutive frames there would be no much change in the position of lane lines with respect to previous images. Moreover also helps is in case of some noise present in the image and histogram leads us to wrong initial seed.

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

The radius of curvature is calculated in the function `find_curvature(...)` function. I passed on the left and right lanes x and y coordinates. The function did the conversion from pixel to meters taking an assumption that our wraped image looks almost same as described in the lecture. We can safely take conversion factor of 25/720 meters per pixels in y direction and 3.0/700 meters per pixels in x direction. After converting from pixel space to meters scale we apply polynomial fit function on both the corresponding left and right lanes x and y coordinates respectively. Using the radius of curvature formula discussed in the lecture and polynomial coefficients found we calculate the curvature at the y_eval taken as 719.
The calculated radius of curvature for the above test image comes out to be 518.692248711 m 489.916019312 m. The left and right lane angle of curvature confirms also that the lanes are parallel and this also confirms correctness of our whole pipeline.

The position of vehicle with respect to center is also calculated by taking the center of detected left and right lane in x axis and subtract that from the center of image. If the value if negative the vehicle is on the left of the center and vice versa. The code is shown only in video pipeline function `process_frame(...)` with comment #Find the position of vehicle with respect to center.


#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step in function `plot_result()`.  Here is an example of my result on a test image:

![alt text][image13]



---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./project_video.mp4)

both are considered so should be included video pipeline overview
For a standout submission, you should follow the suggestion in the lesson to not just search blindly for the lane lines in each frame of video, but rather, once you have a high-confidence detection, use that to inform the search for the position of the lines in subsequent frames of video. For example, if a polynomial fit was found to be robust in the previous frame, then rather than search the entire next frame for the lines, just a window around the previous detection could be searched. This will improve speed and provide a more robust method for rejecting outliers.

For an additional improvement you should implement outlier rejection and use a low-pass filter to smooth the lane detection over frames, meaning add each new detection to a weighted mean of the position of the lines to avoid jitter.

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

Here I'll talk about the approach I took, what techniques I used, what worked and why, where the pipeline might fail and how I might improve it if I were going to pursue this project further.

Where will your pipeline likely fail?
for a long time non presence of any detected line due to failure of parameters and steps used in thresholding techniques
sharp patches on the road which are also parallel to lane Lines and look also similar.
too much noise in the Camera which also contributes towards false thresholding
check points from the guidelines

What could you do to make it more robust?
using the double detection mechanism e.g similar method used on first project of using hough trnasformation and credit Based t decide about the final result
based on the fact that the distance between should remain the same we can use this info both in Look-Ahead Filter and Sanity Check.
improved morphological operation like eroding and dilating
