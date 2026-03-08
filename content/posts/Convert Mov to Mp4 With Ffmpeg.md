---
title: Untitled
description:
date: 2023-03-10T11:28:07-06:00
draft: false
categories:
  - tech
  - life
  - iOS
---
# Convert .mov to .mp4 with ffmpeg

I recently wanted to save a Quicktime screen recording .mov file as .mp4.

There are a number of tools online, but why load any of those when you can just run ffmpeg from your terminal! You can download their installer or on a Mac, just run brew install ffmpeg.

To convert it and also compress the result, you can run:

```Bash
ffmpeg -i my-video.mov -vcodec h264 -acodec mp2 my-video.mp4
```

## If No audio converted
If the converted video has no audio you can try other audio codec options normally aac is a good option to try first. 

```Bash 
ffmpeg -i input.mov -vcodec h264 -acodec aac output.mp4 
```

**✘ Made by Tracer for Pocket Operators  [pocket operators](https://tracer.cc) ✘** 
