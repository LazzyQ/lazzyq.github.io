---
title: "Linux常用工具使用"
date: 2021-01-23T15:18:31+08:00
draft: false
original: true
categories: 
  - 经验
tags: 
  - 工具
---


### SSH 无密码登陆

首先在自己的本地机器生成ssh key

```shell
ssh-keygen -t rsa
```

SSH密钥会保存在home目录下的**.ssh/id_rsa**文件中．SSH公钥保存在**.ssh/id_rsa.pub**文件中．

然后将SSH公钥上传到Linux服务器

```shell
ssh-copy-id username@remote-server
```

输入远程用户的密码后，SSH公钥就会自动上传了．SSH公钥保存在远程Linux服务器的**.ssh/authorized_keys**文件中。

<!--more-->

### 服务启动

ubuntu中，服务可以通过systemctl来管理

比如安装完mysql之后就会有mysql.service这个服务

**启动服务**

```
systemctl start mysql.service
```

**关闭服务**

```
systemctl stop mysql.service
```

**重启服务**

```
systemctl restart mysql.service
```

**显示服务的状态**

```
systemctl status mysql.service
```

**在开机时启用服务**

```
systemctl enable mysql.service
```

**在开机时禁用服务**

```
systemctl disable mysql.service
```

**查看服务是否开机启动**

```
systemctl is-enabled mysql.service
```

**查看已启动的服务列表**

```
systemctl list-unit-files|grep enabled
```

### 判断终端是否走了代理服务器

一行命令搞定：

已翻墙

```
$ curl cip.cc
IP	: 149.129.75.xxx
地址	: 中国  香港  阿里云

数据二	: 香港 | 阿里云

数据三	: 中国香港 | 阿里云
```

未翻墙

```
$ curl cip.cc
IP	: 222.209.33.xxx
地址	: 中国  四川  成都
运营商	: 电信

数据二	: 四川省成都市 | 电信

数据三	: 中国四川省成都市 | 电信
```

### 合并视频和音频

最近下载了Youtube上面的视频

```
format code  extension  resolution note
249          webm       audio only tiny   54k , opus @ 50k (48000Hz), 12.73MiB
250          webm       audio only tiny   69k , opus @ 70k (48000Hz), 16.60MiB
140          m4a        audio only tiny  129k , m4a_dash container, mp4a.40.2@128k (44100Hz), 33.56MiB
251          webm       audio only tiny  132k , opus @160k (48000Hz), 32.78MiB
160          mp4        256x144    144p   64k , avc1.4d400c, 24fps, video only, 11.00MiB
278          webm       256x144    144p   96k , webm container, vp9, 24fps, video only, 23.29MiB
133          mp4        426x240    240p  124k , avc1.4d4015, 24fps, video only, 19.61MiB
242          webm       426x240    240p  188k , vp9, 24fps, video only, 35.61MiB
243          webm       640x360    360p  373k , vp9, 24fps, video only, 63.51MiB
134          mp4        640x360    360p  503k , avc1.4d401e, 24fps, video only, 47.47MiB
244          webm       854x480    480p  758k , vp9, 24fps, video only, 105.52MiB
135          mp4        854x480    480p 1185k , avc1.4d401e, 24fps, video only, 92.02MiB
247          webm       1280x720   720p 1751k , vp9, 24fps, video only, 207.76MiB
136          mp4        1280x720   720p 2872k , avc1.4d401f, 24fps, video only, 174.77MiB
248          webm       1920x1080  1080p 3076k , vp9, 24fps, video only, 394.94MiB
137          mp4        1920x1080  1080p 6553k , avc1.640028, 24fps, video only, 349.89MiB
18           mp4        640x360    360p  411k , avc1.42001E, 24fps, mp4a.40.2@ 96k (44100Hz), 108.63MiB
22           mp4        1280x720   720p  788k , avc1.64001F, 24fps, mp4a.40.2@192k (44100Hz) (best)
```
可以看到，youtube的一个视频提供了很多格式，我当然是下载质量最好的是吧，于是把format是137的视频下载下来，然而发现没有声音，这才发现137是`video only`，没办法再把音频文件140下载现在。然后要做的就是合并这2个文件。
我首先想到的就是pr，于是pr走起，导入视频、音频素材，然后开始导出，发现导出的最终视频2G，原视频才350MB，音频33MB，可能是pr导出时参数设置的问题，无奈我不是很懂，于是想到之前搞过ffmpeg，于是尝试了以下，果然真香

```
ffmpeg -i audio.mp4 -i video.mp4 -acodec copy -vcodec copy output.mp4
```

最终导出的视频400MB，而且速度贼开，2s不到吧

### youtube-dl下载视频

下载所有的字幕和最好的视频

```
youtube-dl --proxy 'socks5://127.0.0.1:1080' --all-subs -f best
```

* --proxy 指定代理
* --all-subs 下载所有的字幕
* -f 指定下载视频的格式，可以通过-F来查看所有的格式

查看视频格式

```
youtube-dl --proxy 'socks5://127.0.0.1:1080' -F 
```

下载播放列表

```
youtube-dl --proxy 'socks5://127.0.0.1:1080' --yes-playlist -f best
```

下载超过100个视频的播放列表

由于目前youtube-dl对于超过100个视频的播放列表会报错[issues 25768](https://github.com/ytdl-org/youtube-dl/issues/25768) [issues 14076](https://github.com/ytdl-org/youtube-dl/issues/14076)，只能通过指定下载列表的[start,end]进行下载

```
youtube-dl --proxy 'socks5://127.0.0.1:1080' --yes-playlist --playlist-start 1 --playlist-end 100 -f best
```

选择最好的格式下载

```
$ youtube-dl --proxy 'socks5://127.0.0.1:1080' -f bestvideo[ext=mp4]+bestaudio[ext=m4a]/bestvideo+bestaudio --merge-output-format mp4 
```

### brew

#### 执行brew命令出现一堆 Warning: Cask 'xxx' is unreadable: undefined method `method_missing_message' for Utils:Module

Firstly, update the local formula repositories to the latest state. Cause reading issues on GitHub repo Hombrew-cask, the error may be introduced by typos in formula definitions.

```sh
# enable --verbose to get more info
brew update --verbose
```

Then try `brew search metabase` again.

If the above command doesn't fix your problem, go into local repository of Homebrew-cask and reset it.

```sh
cd "$(brew --repo homebrew/cask)"
git clean -dfx

git reset --hard origin/master
git pull origin master
```

参考[原回答](https://apple.stackexchange.com/questions/371785/warning-cask-xxx-is-unreadable-undefined-method-method-missing-message-for)

### apt源中有一些google的源，导致 apt-get update无法连接

可以给apt-get设置代理

```
sudo apt-get -o Acquire::http::proxy="http://127.0.0.1:8086" update
```

### mac无线鼠标移动慢

- 鼠标双击阈值：defaults read -g com.apple.mouse.doubleClickThreshold
- 鼠标加速度：defaults read -g com.apple.mouse.scaling
- 滚动速度：defaults read -g com.apple.scrollwheel.scaling

如果鼠标使用有异常，可以再终端中读以上三个参数，并根据自己的需要适当调高调低

- 鼠标双击阈值：defaults write -g com.apple.mouse.doubleClickThreshold 0.75
- 鼠标加速度：defaults write -g com.apple.mouse.scaling 5
- 滚动速度：defaults write -g com.apple.scrollwheel.scaling 0.75