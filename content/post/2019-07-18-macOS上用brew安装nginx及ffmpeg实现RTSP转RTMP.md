---
title: macOS上用brew安装nginx及ffmpeg实现RTSP转RTMP
categories:
  - macOS
date: 2019-07-18 22:15:24
updated: 2019-07-18 22:15:24
tags: 
  - macOS
  - ffmpeg
  - brew
---


# 安装ffmpeg

```sh
brew install ffmpeg --with-ffplay
```

ffplay 是一个播放器，可以直接播放各种流。

# 支持RTMP的Nginx

```sh
brew tap denji/nginx
brew install nginx-full --with-rtmp-module
brew info nginx-full
```
https://www.jianshu.com/p/cf74a34af15d

# 推流

```sh
ffmpeg -re -rtsp_transport tcp -i "rtsp://host/dss/monitor/params?cameraid=1000025%2412&substream=1" -f flv -vcodec libx264 -vprofile baseline -acodec aac -ar 44100 -strict -2 -ac 1 -f flv -s 1280x720 -q 10 "rtmp://localhost:1935/mylive/1"
```

<!--more-->
# Nginx 官方配置

https://github.com/arut/nginx-rtmp-module

# 关于使用微信小程序 IOS 无法播放的问题

这个问题同时在使用的时候用到，安卓是正常的，IOS就是不行，查看了一下后台，连流都没有拉起来，参考此问题的描述：[live-player iOS端解码失败 安卓正常](https://developers.weixin.qq.com/community/develop/doc/00046825fd89e02188584eef157000)

最后是用 ffmpeg 进行推流的时候需要设置一下 flv  的 Profile 与 level 。

[这篇文章详细的描述](https://blog.csdn.net/achang21/article/details/77824485) IOS 支持的设置。

# 关于动态推流的设置(nginx+RTMP)

基本原理是 nginx 开启 rtmp 的支持，当访问 rtmp 链接的时候，执行外挂进行，进行推流命令，就行流的发布。

也就是说在 nginx  的配置中配置一下外挂 shell 脚本，在 shell 脚本中进行 ffmepg 的推流操作。


## nginx.com

```conf
rtmp {
    server {
        listen 1935;
        application live {
            live on;
            exec_options on;
            exec_kill_signal term;
            exec_pull pull_rtsp_stream.sh $app $name;
        }
    }
}
```


```sh
#!/bin/bash
##################################################
#
# nginx build shell
# created by hzm
# 2015.12.21
#
##################################################

mgw_ip="127.0.0.1"
mgw_port=9090
log_file="./../logs/pull_rtsp_stream.log"
app=$1
name=$2

on_die()
{
    echo "channel:$name receive term signal, kill pid:$$ child_pid:$!" >> $log_file
    kill -2 $!
}

echo "mgw_ip:$mgw_ip, mgw_port:$mgw_port,app:${app} name:${name}" >> $log_file

export LD_LIBRARY_PATH=./
(./ffmpeg  -i "rtsp://${mgw_ip}:${mgw_port}/dss/monitor/param?${name}" -profile:v baseline -level 3.0 -vcodec copy -acodec copy  -f flv "rtmp://127.0.0.1:1935/${app}/${name}")&


trap 'on_die' TERM
wait

echo "channel: ${name} exit" >> $log_file

exit 0
```
