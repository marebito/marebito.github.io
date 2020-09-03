---
title: MP4启用-movflags_faststart
date: 2020-09-03 14:10:58
tags: - ffmpeg
	  - mp4
---



你的命令缺少一个加号,它应该让你在输出中“将moov原子移动到文件的开头”.正确的命令是：

```
ffmpeg -i C:\vidtests\Wildlife.mp4 -movflags +faststart C:\vidtests\Wildlife_fs.mp4
```

作为经验法则,当不编码/转码时,我建议使用“-vcodec copy -acodec copy”参数,如下所示：

```
ffmpeg -i C:\vidtests\Wildlife.mp4 -vcodec copy -acodec copy -movflags +faststart C:\vidtests\Wildlife_fs.mp4
```