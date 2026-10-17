---
name: add_music_to_video
description: 目标是给视频添加背景音乐，提升视频的观赏性和感染力。根据给定的视频文件、音乐文件用ffmpeg进行处理，输出带有背景音乐的视频文件。
version: 1.0.1
---

## 工作流程

### 步骤1
检查ffmpeg是否安装在系统中，如果没有安装，提示用户安装ffmpeg，注意不同环境的判断命令行的区别。

### 步骤2
根据用户的promt，确定视频文件和音乐文件的路径和格式，并检查文件是否存在和格式是否正确。
同时解析其他参数，包括音量、视频开始时间、视频结束时间、音乐其实时间、音乐结束时间、开始音乐渐入时长、结束音乐渐出时长等。
其中视频开始时间默认是0，视频结束时间默认是视频文件的总时长，音乐开始时间默认是0，音乐结束时间默认是音乐文件的总时长，开始音乐渐入时长默认是0，结束音乐渐出时长默认是0。

### 步骤3
使用ffmpeg命令将音乐文件与视频文件进行合并，根据解析的参数调整音频和视频的同步、音量等属性，生成最终的视频文件。

### 步骤4
检测原视频是否有音轨，如果原视频有音轨，则需要将原视频的音轨去掉，只保留新的视频音轨。

### 步骤5
检查音频文件音乐时长和

### 步骤5
将步骤2和步骤3中解析的参数和生成的ffmpeg命令行，并且开始执行ffmpeg命令行，处理视频文件。
比如：
Prompt: 
给视频添加背景音乐，视频文件路径为/Users/yanjun/Workspace/github/PythonCourse/media/videos/header1/1300p30/AnimateHeader.mp4，音乐文件路径为/Users/yanjun/Workspace/github/MathArt/videos/Epizykloid_Torus_II_20260915-120500.mp4，视频开始时间为0秒，视频结束时间为6秒，音乐开始时间为0秒，音乐结束时间为6秒，开始音乐渐入时长为2秒，结束音乐渐出时长为2秒。

输出格式
```bash
ffmpeg -i /Users/yanjun/Workspace/github/PythonCourse/media/videos/header1/1300p30/AnimateHeader.mp4 -ss 0 -t 6 -i /Users/yanjun/Workspace/github/MathArt/videos/Epizykloid_Torus_II_20260915-120500.mp4 -filter_complex "[1]scale=iw/2:ih/2[pip];[0][pip]overlay=(W-w)/2:(H-h)/2+350" /Users/yanjun/Workspace/github/PythonCourse/media/videos/header1/1300p30/AnimateHeader_ready.mp4
```

### 步骤6
输出处理结果是一个新的视频文件，视频文件默认放到和原视频相同的目录下，文件名为原视频文件名加上后缀"_with_music"。如果用户指定了输出路径，则将文件保存到指定路径，然后给于用户提示处理完成，并提供输出文件的路径。
如果视频文件已存在，则在输出文件名后加上时间戳，避免覆盖原视频文件。