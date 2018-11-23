# FFmpeg 工具使用

### MAC 版下载安装
http://ffmpeg.org/download.html

### 下载dmg文件，进入dmg的目录将ffmpeg的文件拷出来，直接可以使用

### 使用命令
```
测试程序安装成功
./ffmpeg -h 

切割音频文件:
./ffmpeg -i [url] -ar 16000 -ac 1 -vn -acodec copy -ss [开始时间] -t [时长] [目标位置]

将mp3 转成wav 格式
./ffmpeg -i ../2_01.mp3 -f wav output.wav

转成采样率 16000 单通道的wav文件
./ffmpeg -i ../media_5.wav -ar 16000 -ac 1 -f wav media_5.wav


```

