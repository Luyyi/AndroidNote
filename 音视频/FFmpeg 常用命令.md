# FFmpeg 常用命令

### 1. RxFFmpeg

github地址：https://github.com/microshow/RxFFmpeg

包含常用音视频编辑功能。
可用RxJava2的形式调用。

### 2. 用过的命令

##### ① 压缩视频

```
ffmpeg -i input.mp4 -s 1920x1080 -b:v 1M  -r 20 output.mp4
```

- `-i input.mp4` 原视频

- `-s 1920x1080` 设置分辨率为1920x1080

- `-b:v 1M` 设置视频码率为1Mb/s

- `-b:a` 设置音频码率

- `-r 20` 设置帧率20fps

</br>
##### ② 图片合成视频
```
ffmpeg -y -framerate 10 -f image2 -i inputDir/img_%d.jpg -c:v libx264 -r 10 -pix_fmt yuv420p -vf scale=trunc(iw/2)*2:trunc(ih/2*2) output.mp4
```

- `-framerate 10` 设置输入帧率为10，10张图片/s

- `-f image2` 输入是图片格式

- `-i inputDir/img_%d.jpg` 指定输入文件，图片img_%d命名，%d从1递增

- `-c:v libx264` 指定编码器

- `-r 10` 设置输出帧率

- `-pix_fmt yuv420p` yuv数据格式

- `-vf scale=trunc(iw/2)*2:trunc(ih/2*2)` 设置缩放，这里是把视频的宽高强制设成偶数，否则可能报错“width not divisible by 2”

</br>
##### ③ 拼接两个视频
```
ffmpeg -i first.mp4 -i second.mp4 -f lavfi -i color=0x1D1D1D -filter_complex [0:v]scale=iw:-1[v0];[1:v]scale=iw:-1,pad=w:h:(w-iw)/2:(h-ih)/2:0x1D1D1D[v1];[v0][v1]concat=n=2:v=1:a=0[v] -vcodec libx264 -map [v] output.mp4
```

- `-i first.mp4 -i second.mp4` 设置两个输入视频

- `-f lavfi` 输入视频格式

- `-i color=0x1D1D1D` 设置视频底色

- `-filter_complex` 复杂滤镜
    - `[0:v][1:v]` 表示取第一个文件的视频和第二个文件的视频
    
    - `[v0][v1]` 表示输出的别称
    
    - `[0:v]scale=iw:-1[v0];` 表示视频1保持原来的大小
   
    - `[1:v]scale=iw:-1,pad=w:h:(w-iw)/2:(h-ih)/2:0x1D1D1D[v1];`表示视频2保持原来的大小，计算与视频1相比未被填充地方，指定底色填充
    
    - `[v0][v1]concat=n=2:v=1:a=0[v]`
        - `concat=`表示将输出`[v0]`和`[v1]`拼接起来
        - `n=2:v=1:a=0` 表示共有`n(v+a)`个输入，有`v+a`个输出
        
</br>
##### ④ 视频中提取音频
```
ffmpeg -i input.mp4 -acodec copy -vn output.mp3
```

- `-acodec`