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

[image1]: ./output_images/undistorted_sample_output.jpg "Undistorted"
[image2]: ./output_images/undistorted_sample_output.jpg "Road Transformed"
[image3]: ./output_images/sobelX_sobely_sample_output.jpg "Binary Example"
[image4]: ./output_images/magnitude_thresholding_sample_output.jpg "Binary Exampl"
[image5]: ./output_images/direction_thresholding_sample_output.jpg "Binary Exampl"
[image6]: ./output_images/Threshold_compination_sample_output.jpg "Binary Exampl"
[image7]: ./output_images/Color_thresholding_sample_output.jpg "Binary Exampl"
[image8]: ./output_images/Color_gradient_compination_sample_output.jpg "Binary Exampl"
[image9]: ./output_images/Birds_Eye_View_sample_output.jpg "warped Exampl"
[image10]: ./output_images/Polynomial_Fitting_sample_output.png "line_fit Exampl"
[image11]: ./output_images/Final_result_sample_output.png "final Exampl"

[video1]: ./project_video.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!

### Camera Calibration

#### 1. Caliberate the camera using (9*6) chessboard to estimate the camera parameters (intrensics and extrisics).



I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.<br>
I made `Caliberate_camera(path_to_imgs,chess_board = (6,9))` function which aliberates the camera and gets the camera parameters used for image undistortion
I made `undistort_img(img,mtx,dist)` which uses `cv2.undistort()` to correct the image distortion and obtained this result:

![alt text][image1]

### Pipeline (single images)

#### 1. Correcting the distortion of the image using camera parameters obtained from camera caliberation

I applied  `undistort_img(img,mtx,dist)` function to correct the distortion of each image before starting any processing on image and here's sample output:
![alt text][image2]

#### 2. Applying Color and Gradient thresholding and try to identify the lane lines

I seperate this step into three phases:<br>
##### a. Applying gradient , magnitude and direction thresholding to an image and then compine them together.
##### b. Aplying color thresholding with HLS , and compine H and S channel together
##### c. Compining the output of first and second phases together to get the best identification of lane lines.

###### a. Steps of the first Phase : <br>
I used `gradient_threshold(img, orient='x', thresh_min=0, thresh_max=255)` which generates an output binary image by applying image gradient to the image in either direction  and here's the sample output of this function:
![alt text][image3]

I used `mag_thresh(img, sobel_kernel=3, mag_thresh=(0, 255))` which generates an output binary image by applying image gradient to the image in both direction to get the magnitude of the gradient and here's the sample output of this function:
![alt text][image4]

I used `dir_threshold(img, sobel_kernel=3, thresh=(0, np.pi/2))` which generates an output binary image by applying threshold on the direction of the image gradient and here's the sample output of this function:
![alt text][image5]

I used `generate_threshold_compination(sobel_x,sobel_y,magnitude,direction)` which generates a combination of all image thresholding techniques and here's the final output of first phase # (a)
![alt text][image6]


###### b. Steps of the second Phase : <br>

I used `color_thresholding(img)` Plays with color channels (HLS) , generates binary compined output of color channels (H and S) and here's the output of this phase :  
![alt text][image7]

###### c. Steps of the Third Phase : <br>

Compining the output of the first and second phase together using `generate_binary_compined(color,combined_threshold)` which generates a binary combination between color thresholding and gradient thresholding and here's the final output of thresholding techniques:
![alt text][image8]


#### 3. Performing prespective transformation to and image.

The code for my perspective transform includes a function called `Bird_eye_view()`.  The `Bird_eye_view()` function takes as inputs an image (`img`) and performs prespective transformation to an image to get bird's eye view with respect to image midpoint:

```python
    upper_left_pt  = [mid_point-90,465]
    upper_right_pt = [mid_point+90,465]
    lower_left_pt  = [mid_point-380,image.shape[0]]
    lower_right_pt = [mid_point+380,image.shape[0]]
    
    new_upper_left_pt  = [300,213]
    new_upper_right_pt = [1050,213]
    new_lower_left_pt  = [350,image.shape[0]]
    new_lower_right_pt = [933,image.shape[0]]
```

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][image9]

#### 4. Identifying lane pixels and fit polynomial lines.

Given the warped binary image from the previous step, I now fit a 2nd order polynomial to both left and right lane lines. In particular, I perform the following:

- Calculate a histogram of the bottom half of the image
- Partition the image into 9 horizontal slices
- Starting from the bottom slice, enclose a 50 pixel wide window around the left peak and right peak of the histogram (split the histogram in half vertically)
- Go up the horizontal window slices to find pixels that are likely to be part of the left and right lanes, recentering the sliding windows opportunistically
Given 2 groups of pixels (left and right lane line candidate pixels), fit a 2nd order polynomial to each group, which represents the estimated left and right lane lines

and here's the output of this phase:

![alt text][image10]

#### 5. Calculating radius of curvature and car position .

I used `curvature_estimation(left_fit,right_fit,ploty)` which calculates the radius of curvature in meters applies this equation (1+(2Ay+B)^2)^(3/2) /|2A|
I used ` position_estimation(warped,left_fit,right_fit)` which estimates the car position with respect to center

#### 6. Reprojection to the image.

I used `draw(undist,warped,left_fitx,right_fitx,ploty,Minv,left_curverad,right_curverad,offset)` which reproject the detected region back on the original image , drawing the left ,right curvature and offset values 

and here's the final result:

![alt text][image11]

---

### Pipeline (video)

#### 1. Last step is to apply the pipeline on the video :
Here's a [link to my video result](./project_video_output.mp4)

---

### Discussion

#### 1. problems and issues
The lane-finding will fail when the road have cracks and could be mistaken as a lane-line (see 'challenge_video.mp4') 
Here's a [link to my video result](./project_challenge_video_output.mp4)

As a fututre work  we can use deep learning to segment the road and lane-lines instead of using classical ways of image processing techniques.
