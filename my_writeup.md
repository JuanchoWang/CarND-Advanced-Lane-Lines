# **Advanced Lane Finding** 

## My Writeup for Submission

#### Hereby I describe how I finish the 2nd project of this Nanodegree programme and how I achieve different goals. I have processed all the test images with the IPython notebook file ["./examples/example.ipynb"], while all the video files were processed with another IPython notebook file ["./video_process.ipynb"]. In the following description, I will almost only refer to the code in the latter file, which is more concise.

---

**Created on 12.05.2020**
First writeup finished.

[//]: # (Image References)

[image1]: ./output_images/cal_img_calibration1.jpg

---

### Camera Calibration

#### Before starting this pipeline, the to-be-used camera was calibrated with a set of classical chessboard images to get the camera intrinsic parameters and distortion parameters.

In the 2nd cell of the `video_process.ipynb`, a function "camera_calibrate" is defined and is designed to find the corners in the chess board images and return the camera intrinsic parameter matrix and distortion parameters. Numbers of (inside) corners in both horizontal and vertical directions on the chess board are input arguments as prior knowledge. Assume that all the corners are on the same plane in one world coordinates system. There are totally 20 samples for calibration and in each sample image, the corners have different image coordinates. The Opencv function `cv2.findChessboardCorners` is applied to find those image coordinates and then all the pairs of image and world coordinates contribute to find a proper set of camera matrix, distortion, displacement and rotation vectors with function `cv2.calibrateCamera`. **Note that at my side, the `cv2.findChessboardCorners` failed to find the corners in three of the calibration images and they were not used for the further calibration** One undistorted chessboard image:

![alt text][image1]
