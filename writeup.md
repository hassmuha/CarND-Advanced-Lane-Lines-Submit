
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

[image1]: ./examples/original_image.jpg "Checkerboard Original Image"
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

The code for this step is contained in the first code cell of the IPython notebook located in "./main.ipynb" .  

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result:

![alt text][image1]

Undistorted Image
![alt text][image2]

### Pipeline (single images)

let's start with test1 image as shown below:

![alt text][image3]

#### 1. Provide an example of a distortion-corrected image.

I have defined a new funciton so that same function can be used for processing video frames as well. This is defined as `distort_correct()` which is using the opencv function `cv2.undistort()` and calibration and distortion coefficients as describe above to calculate the undistorted test image. The undistorted test1 image is shown below:

![alt text][image4]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I used a combination of color and gradient thresholds to generate a binary image.  I used following steps as present in the function `pipeline()` :

1) Color space conversion : RGB to HLS
2) Applying gaussian smoothing on L plane to prepare to calculate gradient on that plane. The video seems already blurred so I used the mask of 3 x 3.
3) Apply x-gradient with Sobel mask of size of 5x5 on L-plane. The L plane is selected as it contain major features of the whole scene along with presence of clear edges. Based on the fact that we are interested on lane lines x-gradient was enough. The threshold image `gradx_L` is calculated by applying thresholding on the absolute of gradient image with values between 20 to 100 are considered as white. This is shown in the figure below:

![alt text][image5]

4) Simple thresholding is applied on S-plane with values between 140 to 255 taken as white. The reason is it will help to detect yellow and clear white lines if present in the image while not being effected by different change in lighting conditions. The resulted image is store in the variable `s_binary` and picture is shown below

![alt text][image6]

5) Simple thresholding is applied on H-plane with values between (15, 40) taken as white. The result shown below also shows it will also be able to detect yellow and white lines. The resulted image is store in the variable `h_binary`:

![alt text][image7]

6) Then combine `s_binary` and `h_binary` with 'and' operation. this reduces a lot of noise and make sure pure lane lines will be detected. The resulted image is stored in the variable `combined_sh` and resulted image is shown below:
![alt text][image8]


7) Then combine `combined_sh` with `gradx_L` with 'or' operation. Mostly lane lines near to the camera were clearly identified by the H and S plane thresholded images, while the further far lane line with curvature information is present in `gradx_L` image. That is the reason of using or operation. Here's an example of my output for this step:

![alt text][image9]

7) At the end Region of Interest based thresholding technique is also applied to remove much background and left with road views

![alt text][image10]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

For taking the perspective transformation I selected four points on the straight road images provided and these points form more or less a shape of trapezoid. As a destination I mapped it to a rectangle. The code is shown in the section 'Apply a perspective transform to rectify binary image'.

This resulted in the following source trapezoid and destination rectangle points are given in the table:

| Source        | Destination   |
|:-------------:|:-------------:|
| 585, 460      | 320, 0        |
| 203, 720      | 320, 720      |
| 1127, 720     | 960, 720      |
| 695, 460      | 960, 0        |

Moreover  `cv2.getPerspectiveTransform(src,dst)` function is used to get the perspective transformation and defined the my own `wrap_perspectTransform()` which takes the perspective matrix and the image as input and using `cv2.warpPerspective()` transformed the input image and returns the warped image as output. For showing the results, I am taking the undistorted threshold image as input and after transformation warped image is shown below:

![alt text][image11]


The lane lines appear parallel in the warped image, this provide us confidence that the transformation is working fine and later using the angle of curvature we can also confirm this.


#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

I have used the two functions as provided in the lecture `slidingwindow_polyfit(...)` and `lookAhead_polyfit(...)`
Function `slidingwindow_polyfit(...)` uses the technique of partitioning the whole image into segments along y axis. Then find possible location of left and right plane starting from the bottom segment and moving up till the first segment.
For the first segment histogram is calculated along x-axis. Then pass a separate window search to find peaks of the histogram which are parts of left and right lane. The final position of window in each segment also serves as starting point to find the left and right lane in next upper segment, so called seed. Widow is move left and right to find best peak. And keep applying these steps till the top segment of the image. For each segment and for each lane I took center of the window points to fit 2nd order polynomial function. This is done in by using `numpy.polyfit(...)` function.

In the figure below the detected widow peak is shown along with curve line is also drawn which is calculated based on the polynomial coefficients found :

![alt text][image12]

But for the video pipeline `lookAhead_polyfit(...)` is also used, this step will be explained later but main idea here is not to perform the sliding window step to avoid computation and exploit the temporal fact that in two consecutive frames there would be no much change in the position of lane lines with respect to previous images. Moreover this also helps in case of some noise present in the image and histogram leads to wrong initial seed.

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

Here's a [link to my video result][video1]


For the frame, the pipeline is present in the function `process_frame(...)`. I have also uses the sanity check to find the relation of probable detected line with lane lines detected in previous frames. Moreover Look-Ahead Filter approach is also used to get advantage from the fact, each corresponding frame contains similar traits as previous frames and no need to blindly start searching of lane lines within frame. Other feather like suggested e.g. smoothing of lane lines before drawing and Reset of previous history in case of Sanity check fails for consecutive frames.


---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?
Listed below are some of the problems which I have faced during the implementation of the project:
- Generation of binary images and which plane to consider for gradient and for which planes only thresholding works well. Moreover the selection of threshold parameters for different planes
- Selection of source and destination points for perspective transformation.
- Defining a good sanity check algorithm when to accept lane lines from the current frame and when to reject it.

I believe the pipeline is vulnerable in following cases:
- For a long time non presence of any detected lines due to failure in extracting lane lines in gradient and thresholded binary images
- Sharp patches on the road which are also parallel to lane Lines and look also similar.
- Too much noise in the Camera which also contributes towards wrongly selected thresholding parameters
- Sharp turns along with fast speed of car

If more time permit I am interested in following improvements to make it more robust:
- Using the multiple detection mechanism e.g similar method used in first project of using hough transformation along with the currently implemented algorithm. At the end apply credit-based mechanism to decide which lane line to consider if both algorithms give different results
- Improved morphological operation like eroding and dilating applied on binary image
- Improved sanity check and use prediction mechanism based on the other road surrounding. main goal will be never to go off the road in any case.
- Use machine learning approaches as well beside conventional image processing for example what we did in the Behavioral Cloning Project.
