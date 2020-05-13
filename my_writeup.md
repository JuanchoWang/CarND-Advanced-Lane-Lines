# **Advanced Lane Finding** 

## My Writeup for Submission

#### Hereby I describe how I finish the 2nd project of this Nanodegree programme and how I achieve different goals. I have processed all the test images with the existing IPython notebook file [./examples/example.ipynb](./examples/example.ipynb), while all the video files were processed with a new file [./video_process.ipynb](./video_process.ipynb). In the following description, I will almost only refer to the code in the latter file.

---

**Created on 12.05.2020**
First writeup finished.

[//]: # (Image References)

[image1]: ./output_images/cal_img_calibration1.jpg

[image2]: ./output_images/cal_img_test2.jpg

[image3]: ./output_images/stacked_bin_test3.jpg

[image4]: ./output_images/mag_S_thresholded_test3.jpg

[image5]: ./output_images/labeled_straight_lines1.jpg

[image6]: ./output_images/warped_straight_lines1.jpg

[image7]: ./output_images/fitted_bin_test2.jpg

[image8]: ./output_images/result_test3.jpg

---

### Camera Calibration

#### Before starting this pipeline, the to-be-used camera was calibrated with a set of classical chessboard images to get the camera intrinsic parameters and distortion parameters.

In the 2nd cell of the `video_process.ipynb`, a function "camera_calibrate" is defined and is designed to find the corners in the chess board images and return the camera intrinsic parameter matrix and distortion parameters. Numbers of (inside) corners in both horizontal and vertical directions on the chess board are input arguments as prior knowledge. Assume that all the corners are on the same plane in one world coordinates system. There are totally 20 samples for calibration and in each sample image, the corners have different image coordinates. 

The Opencv function `cv2.findChessboardCorners` is applied to find those image coordinates and then all the pairs of image and world coordinates contribute to find a proper set of camera matrix, distortion, displacement and rotation vectors with function `cv2.calibrateCamera`. **Note that at my side, the `cv2.findChessboardCorners` failed to find the corners in three of the calibration images and they were not used for the further calibration** One undistorted chessboard image:

![alt text][image1]

---

## Pipeline

In `video_process.ipynb`, all the following steps are either written compactly in one function `process_image` in 3rd cell or defined as individual functions in 4th cell but called by the previous function.

### 1. Undistortion

#### With camera calibration information, the original image/frame is distortion-corrected

This step is simply implemented with function `cv2.undistort` at the beginning of the 3rd cell. One undistorted road image:

![alt text][image2]

### 2. Thresholding Binary Image

#### Applying color transform, gradient magnitude to the image along with thresholding, the pixel candidates for lane can be represented by binary images.

I here only use gradient magnitude and S-channel of HLS color space. After converting the image to gray scale, I apply sobel operator to it for both x and y directions. The magnitude of gradient is then calculated and scaled to 0-255 for the entire image. Meanwhile, the RGB image is also converted to HLS and only S-channel is extracted and scaled. I set the threshold for both as:

```python
mag_threshold = (60, 255)
S_threshold = (100, 255)
```

I stacked both thresholded binary images to 3 channels as a RGB-image to preview it:

![alt text][image3]

A logic "OR" operation is applied to combine those two binary images and the result is:

![alt text][image4]

### 3. Perspective Transform

#### Given a selected area on the current lane ahead in the image, the camera view is transformed to a top-down view, in which both lane lines look roughly parallel.

To specify this area, I chose a trapezoid-shape area from a test image with straight lane lines and the vertices are hard-coded. In fact, I manually marked 4 points in the image and then got their coordinates. See below.

![alt text][image5]

The relevant coordinates will be used for `cv2.getPerspectiveTransform` to determine a transformation matrix as below:

```python
label_img_coords = [[573, 467], [709, 467], [1007, 667], [293, 667]]

offset_values = (300, 0)
offset_x = offset_values[0]
offset_y = offset_values[1]
topdown_coords = [[offset_x, offset_y], 
                  [img_size[0] - offset_x, offset_y], 
                  [img_size[0] - offset_x, img_size[1] - offset_y],
                  [offset_x, img_size[1] - offset_y]]
                  
src_pts = np.float32(label_img_coords)
dst_pts = np.float32(topdown_coords)
pt_mtx = cv2.getPerspectiveTransform(src_pts, dst_pts)
```

Afterwards, this matrix is utilized to warp the image for perspective transformation with the function `cv2.warpPerspective`. See the warped image:

![alt text][image6]

### 4. Identifying the Lane Line Pixels & Fitting with a Polynomial

#### Using sliding windows method, valid lane pixel candidates are identified from warped binary images. Next, they will be fitted to two lane lines by polynomial with respect to pixel.

This step is mainly done in two functions `find_lane_pixels` and `fit_polynomial` in the 4th cell. As a starting point of sliding window method, a pixel number histogram is taken from the lower half of the warped thresholded binary images. The maximum positions on the left and right side of middle point are the base points.

The same set of hyperparameters is employed as in the practice. 

```python
# Hyperparameters for finding lane pixels
# Choose the number of sliding windows
nwindows = 9
# Set the width of the windows +/- margin
margin = 100
# Set minimum number of pixels found to recenter window
minpix = 50
```

Each window is a region of interest in the warped binary image. Starting from the base points, those windows will slide vertically from bottom to top for 9 times. With a fixed margin and height, each window has a clear boundary and all the valid pixels are considered as lane line pixels. The horizontal location of the window is based on the lower center points. The lower center points of the two windows at the most bottom are the base points. As long as the valid pixels in the window outnumber a defined threshold, the window for the next slide has to be re-centered according to mean position. In this way, almost all the lane pixels can be identified.

If all the lane line pixels are found, two lane lines can be easily fitted with 2nd order polynomials using function `np.polyfit`. For visualization, the lane line pixels on the left are marked in red, the ones on the right in blue, two fitted lane lines are thin lines in yellow and the boundaries of each sliding window are displayed in green.

![alt text][image7]

### 5. Curvature Radius & Deviation from Center

#### The lane lines should be fitted again with respect to meter and their curvature radius at the current vehicle position can be calculated according to a formula.

To fit the lane pixels over meters, it must be clear how long in meters in reality are equivalent to one pixel in the image. I first defined a function `coordTransfrom` in the 4th cell. This function is based on the basic camera optics and implements a 2D-to-3D coordinate transformation. 

Unfortunately, due to lack of more calibration information, I personally assume that the camera mounting height is 1.4m and there is no pitch, yaw or roll rate. I chose this value as camera height, because I figured out that based on such a configuration, the distance between two lower selected points is roughly 3.7 meters, which is consistent with the width of lane in accordance with US regulations. 

For the next step, I got the longitudinal distance between vehicle and the upper selected points in meters. Their difference is the longitudinal length of my specified area in reality. Finally I could calculate the scaling factors from pixel to meter in both x and y directions:

```python
label_world_long_lenth = np.mean([abs(coordTransfrom(label_img_coords[0], cam_height, mtx)[0] - 
    coordTransfrom(label_img_coords[3], cam_height, mtx)[0]),
    abs(coordTransfrom(label_img_coords[1], cam_height, mtx)[0] - 
    coordTransfrom(label_img_coords[2], cam_height, mtx)[0])])
label_world_lat_length = 3.7  # Just use standard value in US regulations

meter2pix_fac = (label_world_long_lenth / img_size[1], 
                 label_world_lat_length / (img_size[0] - 2*offset_x))
```

With the help of this scaling factor, both lane lines are fitted again over meters. The final curvature radius is a weighted mean value over both sides and the weight is the number of valid pixels:

```python
left_curverad = (1 + (2*left_fit_cr[0]*y_eval + left_fit_cr[1])**2)**1.5 / np.absolute(2*left_fit_cr[0])
right_curverad = (1 + (2*right_fit_cr[0]*y_eval + right_fit_cr[1])**2)**1.5 / np.absolute(2*right_fit_cr[0])
    
mean_curverad = (len(leftx)*left_curverad + len(rightx)*right_curverad) / (len(leftx) + len(rightx))
```

And the vehicle position with respect to center is calculated according to the lowest identified lane pixel:

```python
## Deviation from center ##
x_disp = 640 - (left_fitx[-1] + right_fitx[-1]) / 2
x_disp = xm_per_pix*x_disp
```

### 6. Plotting Results Back to the Road

#### The identified lane pixels and fitted lines are plotted back to camera view with an inverse perspective transformation. The mean curvature radius and deviation will also be displayed.

Just exchange the source points and the destination points for `cv2.getPerspectiveTransform`, the inverse perspective transformation matrix is determined. With it, the result on binary images are plotted back onto the road in camera view. Additionally, the space between two fitted lines are marked in green. Finally, the estimated curvature radius and vehicle horizontal distance from center are displayed in the image with the function `cv2.putText`.

![alt text][image8]

### 7. To make it a video!

#### All the 6 steps above are implemented in order for each frame of the project video and the results make up the output video.

Due that my function `process_image` requires 5 arguments, 4 more than the method `VideoFileClip.fl_image()` expected, I used a lambda function to comply with the requirement:

```python
process_frame = lambda frm: process_image(frm, cal_params, label_img_coords, cam_height, offset_values)
output_clip = clip1.fl_image(process_frame)
```

[Here](./output_videos/project_video_output.mp4) is the link to my output video.

---

## Discussion

### Shortcomings

I also tried to process both challenge videos. However, I couldn't get so good result as from the easy video. I have thought of the following issues:

* During window sliding, some windows have moved to the side boundary of the image, which make the current code fail.
* The sliding window method now is implemented for each frame. Specifically, for each frame, both base points are reset and both lane lines are re-fitted. The temporal context hasn't been utilized.
* The combination of gradient magnitude and S-channel as well as their thresholds are subjectively decided by me. This time, it could fit the project video but it is not suitable for more scenarios.
* Manually selecting the camera moungting height is just like guessing, which often leads to more serious consequences than the manually chosen source points.

### To-Be-Improved

It is quite clear to me that at least the second issue could be corrected. The base points for sliding windows could be derived from the last frame and the fitted line should be smoothed for two consecutive frames.