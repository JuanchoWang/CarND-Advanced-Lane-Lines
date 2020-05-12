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