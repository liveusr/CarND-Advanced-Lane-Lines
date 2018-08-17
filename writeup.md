# **Advanced Lane Finding**

The goal of this project is to build an image processing pipeline that finds lane lines on the road. To start with, code from various assignments and quizes was used. Further experimenting with it, final iage processing pipeline was implemented. This document describes various steps used in the pipeline and the approach used. 

## Camera Calibration
Camera transforms real world 3D objects into a 2D image. This transformation is not perfect. Images captured by camera are distorted. Distortion changes apparent size/shape of an object in the image. We need to undo this distortion so that we can get correct information. We can calculate distortion coefficients by processing images in which size and shape of objects in the image is pre-defined and already known.

### Find Chessboard corners and Calibrate Camera
Images of chessboard pattern are useful for calibrating camera as its high contrast pattern is easy to detect and we know what an undistorted chessboard looks like. "camera_cal" folder contains multiple images of 9x6 chessboard pattern taken at different angles and distances. Coordinates of corners in this 2D image are detected using `cv2.findChessboardCorners()` function - it's called image points. These points are mapped to 3D coordinates of real undistorted corners - called object points, and are same for all calibration images. Z-coordinate will be zero for all obeject points as chessboard is on flat image plane. Following image shows example of detected chessboard corners:

![alt text](writeup_data/chessboard_corners.png "chessboard_corners")

### Calculate Distortion Coefficients
Image points and object points of all calibration images are stored in `imgpoints` and `objpoints` array. With the help of imgpoints and objpoints, we can calibrate camera using `cv2.calibrateCamera()` function. This function returns camera matrix and distortion coefficients. Camera matrix and distortion coefficients are used to undistort the image usig `cv2.undistort()` function. Following image shows example of undistorted chessboard image:

![alt text](writeup_data/chessboard_undistort.png "chessboard_undistort")

## Pipeline (test images)
Image processing pipeline consists of various steps. First we apply distortion correction on an input image. Then we apply various color transforms and gradient filters resulting in a binary image with highlighted/enhanced lane markings eliminating surrounding noise as much possible. This image is then warped so that we can accurately find lane pixels and calculate lane curvature. Following sections explain various experiments performed and steps involved building a final image processing pipeline.

### Distortion Correction
As a very first step, we undistort input image so that we can extract correct and accurate information out of the image. Camera matrix and distortion coefficients calculated above during camera calibration are used. Following image shows undistorted input image:

![alt text](writeup_data/testimage_undistort.png "testimage_undistort")

### Color Transforms and Gradients
I experimented with various color transforms and gradient thresholding for generating a binary image with highlighted and enhanced lane markings with minimal noise. Out of all the color transforms I tried, I found out that S-channel and L-channel of HLS representation and V-channel of HSV representation contained more accurate information about lane markings. Following image shows these three color channels:

![alt text](writeup_data/slv_channels.png "slv_channels")

I applied filters to HLS S-channel and L-channel values, but it has some noise in shadow region. However V-channel could filter solid lane marking really nice. But still dotted lines away from camera were not very clear. So I applied absolute Sobel-x operator with kernel size 15 on HLS L-channel. Taking gradient in x direction emphasizes edges closer to verticle. This helped extract smaller pixels of both solid and dotted lane markings which are away from camera. Then output of both the above operations - HSV V-channel filter and Sobel-x on HSL L-channel is combined to form the binary image with extracted lane line markings. Following image shows output of HSV V-channel filter, Sobel-x on HSL L-channel and the combined filter of these two operations: 

![alt text](writeup_data/processedimage.png "processedimage")

### Perspective Transform
Using perspective transform, camera image is transformed to birds-eye view in such a way that both lane lines look parallel to each other. It gives an effect as if we are looking down on the road from above. Perspective transform maps source image points to a different desired image points with new perspective. After carefully experimenting with source and destination points, following final values of `src` and `dst` points are used:

|src|dst|
|---|---|
|(268, 700)|(320,720)|
|(589, 457)|(320,0)|
|(689, 457)|(980,0)|
|(1065, 700)|(980,720)|
  
This is really useful for calculating the lane curvature and position of the car with respect to left and right lane lines. Perspective transform is applied on the undisorted pre-processed image resulting in a clear and sharp warped image. Following image shows example of warped image: 

![alt text](writeup_data/warped.png "warped")

### Find Lane-Lines Pixels
After applying series of transforms and gradient thresholding, we have a thresholded warped binary image in which lane lines stand out clearly. Now, to find out which pixels are part of the lane lines, histogram along columns is calculated. And we can see peaks in the histogram along the lane lines. 

So starting with bottom of the image and I used two highest peaks from histogram as a starting point where lane lines are, then used sliding windows moving upward to find out where lane lines go. After finding all pixels beonging to each lane line, I fit a second order polynomial to each line - left line and right line. `find_lane_pixels()` function finds the lane pixel using sliding window approach, and `fit_poly()` fits a polynomial.

In the video, we don't really need to run full sliding window algorithm everytime since lines don't move a lot from frame to frame. So, we can search in a margin around the previous detected line position. This is implemented in `search_around_poly()` function. Following image shows output of sliding window and search from prior:

![alt text](writeup_data/fitpoly.png "fitpoly")

### Calculate Radius of Curvature and Vehicle Position
To calculate radius of the curvature, x and y pixel values from pixel world are converted to real world. Lanes projected from camera image are around 30 meters long and 3.7 meters wide in real world. In the warped image they fit in the `dst` rectangle [(320, 0), (980, 720)], i.e. 720 relevant pixels in y-direction and 660 relevant pixels in x-direction. So, following conversion formula is used to convert pixels to real-world meter measurements:
**`ym_per_pix = 30.0/720`** `# meters per pixel in y dimension`
**`xm_per_pix = 3.70/660`** `# meters per pixel in x dimension`
Function `measure_curvature_drift()` calculates radius of road curvature and position of a car with respect to left-right lane lines.

### Identify Lane Area
Once lane lines are detected in warped image, measurements are projected back on to the original road image. For this transformation, inverse perspective matrix is calculated by interchanging `src` and `dst` values. Following images shows radius of curvature and vehicle position overlayed on original image along with marked lane area:

![alt text](writeup_data/measurements.png "measurements")

## Pipeline (video)
The image processing pipeline created above is used to find lane lines in frames from video. Every frame displays identified lane lines, measured radius of curvature and car position within the lane. Following gif shows snippet of output video:

![alt text](writeup_data/output_video.gif "output_video")

Complete output video can be found here: [output_video.mp4](output_video.mp4)

## Challenges Faced / Further Improvements

The most interesting and challenging part of this project was color filtering and and creating gradient threshold. Some transformations were good to detect solid lines, some were good for dotted lines. Some could detect yellow lines really nice where as some detected hite lines. But at the same time they were introducing some noise in the resultant filter. Finding line on cement road and shadowed are was most difficult. Instead of creating one filter that creates both lane lines with some noise, I decided to follow approach where I created different filters which could detect at least some part of lanes without introducing any noise and then combined output of them to form final filter to create thresholded binary image. Combination of HSV V-channel filter and Sobel-x on HLS L-channel proved the best. 

Another difficult part was to create `src` and `dst` coordinates for perspective transform. If the source coordinates are not accurate enough, it introduces lot of noise in warped image, and if the destination coordinates are not accurate, it can introduce errors in measurements as farther pixels zoomed in a lot in warped image if dst is very wide, or zoomed in a very little if dst is very narrow. 

For further improvements, Line class can be implemented that can keep track of past few valid detected frames and their measuments. This can help smoothen the output by eliminating incorrectly calculated measurements. 
