# Using Stable Diffusion for 3D Video Synthesis

Stable Diffusion 2.0 has added support for [depth map](https://huggingface.co/stabilityai/stable-diffusion-2-depth) estimation from static images.  This project documents an experiment to determine if the depth maps are stable enough to convert a 2D video into a 3D video by using AI to synthesizing the second eye viewpoint.  The answer for one test clip is that it works much better than expected and is worth exploring further.

## Stable Diffusion Setup

I started by installing the Stable Diffusion [webui](https://github.com/AUTOMATIC1111/stable-diffusion-webui) on my PC.  Additionally in the webui-user.bat file I set `-xformers` for the speedup as well as `--api` and `--listen` to use the webgui locally.  (I also had to accept connections on port 7860).  Neither `--api` nor `--listen` were actually required to do the 2D to 3D conversion.

Installing the Depth extension was as easy as opening the 'Extensions' tab in the webgui, selecting `Available', and installing 'stable-diffusion-webui-depthmap-script'.  This adds a 'Depth' tab to webgui that works better than expected.

## Depth Maps For Video

To start I converted the target video into separate PNG images with ffmpeg: `ffmpeg -i original.mkv frames/image-%04d.png`.  Note that converting from a video format to PNG frames can use a large amount of disk space as it is not as efficient of a format.

[![Original video](https://img.youtube.com/vi/SfrOwzHhQKU/0.jpg)](https://youtu.be/SfrOwzHhQKU)
[Original source video](video/original.mkv)

The webgui Depth format already supports 'Batch From Directory' so I simply ran it with the image frames and the following option settings:
* Check 'Match input size'
* Check 'Save DepthMap'
* Check 'Generate Stereo side-by-side image'
* Uncheck 'Combine into one image'

Running Stable Diffusion in this way took 262 minutes for 1373 frames of size 1080x1920 on my modest PC.  This turns out to be just over 11.5s per frame and is by far the slowest part of the process.

A test video was made after all of the depth images had been created to provide insight into how stable hte resulting synthesis was across frames.  Stable Diffusion does not receive any frame to frame information so each depth image is independent of the others.  It turns out that there is quite a bit of flicker in the resulting depth map but it is not as noticeable in the resulting 3D video. `ffmpeg -framerate 29.97 -i 'depth/image-%04d-0000.png' -c:v libx265 -pix_fmt yuv420p depth.mpk`

[![Depth video](https://img.youtube.com/vi/R_KGRCTsQVc/0.jpg)](https://youtu.be/R_KGRCTsQVc)
[Depth source video](video/depth.mkv)

Next I reassembled the images back into a 3D video via ffmpeg again: `ffmpeg -framerate 29.97 -i 'depth/image-%04d-0001_stereo.png' -c:v libx265 -pix_fmt yuv420p -metadata:s:v stereo_mode=left_right stereo.mkv`  Note that this reencodes the video using default x265 settings.

[![Stereo video](https://img.youtube.com/vi/wCGxXOy_RDs/0.jpg)](https://youtu.be/wCGxXOy_RDs)
[Stereo source video](video/stereo.mkv)

## Notes

### Fantastic results

The resulting video looks fantastic when played on a 3D headset.  However this was a limited test in that 1) There is only one rather short and limited scene, 2) There is not much camera motion.

### Synthesize non-dominant eye

Synthesize the image for the non-dominant eye.  For most people this will be the left eye but not for everyone.  The results seem very forgiving of depth map noise in part because of this effect.

### Good playback is surprisingly a nuisance

Stereo playback was surprisingly difficult on several devices(Quest 2, PC, phone).
* mkv stereo_mode doesn't work out of box.
* The aspect ratio for Side By Side(SBS) video is awkwardly sometimes 2:1 but not always.
* Youtube fails to play stereo even with yt3d:enable=true set.  It plays an almagram image on PCs but not mobile.  It is not stereo on the Quest 2.
* 'Bigscreen' on Quest 2 displays the video with much manual intervention. It was required to convert the video to mp4: `ffmpeg -i stereo.mkv -aspect 1080:1920 -map 0 -c copy -strict -1 stereo.mp4`

##  Future Projects

### Set up dedicated pipeline

This naive process makes many intermediate copies and conversion of video frames.  It would be faster if the depth estimation worked directly on YUV420 video without the multiple PNG intermediaries.  Additionally all of the images are pushed to disk or network rather than handled in memory.

### Experiments for speeding up depth generation

11.5 seconds per frame is far from real time.  There are likely a bunch of tricks that could be used to speed this up (scaled down depth images, smaller video, interpolate depth between keyframes, grayscale only input images, run 300 webgui in parallel, etc) but getting a 300x speedup would be difficult.

### Frame to frame depth smoothing

The resulting depth video flickers as the estimations are all independent of each other. The resulting 3D video looked surprisingly fantastic but there is room for improvement in smoothing frame to frame depth estimation.  This could be done via averaging, or normalization of the background, or by pulling the video encoder's motion vectors out to determine similarity.  It's not clear whether these are worthwhile vs simply running each frame independently as that is trivial to parallelize.

### Better occlusion handling in the generated images

Interestingly the Stable Diffusion Depth extension is already set up to create side by side images.  It seems to do a naive projection with texture clamping.  There are some areas of the left eye reconstruction which are simply not visible from the right eye and must be created in some other way.  For the sample video this presents itself in the smearing of the rightmost 4 pixels of the left image, as well as occasional smearing on the left side of highly angled objects as they cross the center of the screen.  Synthesizing more plausible filler for these areas is tricky because the results have to be consistent across video frames.

### Simplify use case

Most of the depth map flickering is on the background elements.  This method could work great for talking heads or chat applications where the backgrounds are not as important.

### Upstream the converter as a webui extension

This process could be bundled together into a script or webui extension as it appears to be a viable new addition to the Stable Diffusion use cases.
