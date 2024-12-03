---
title: Linux实用命令手记
date: 2018-10-07 21:22:52
tags: [linux,shell,容器]
category: [计算机技术]
thumbnail: /img/linux-tech-tips/63143834.png
lede: 毛胚房与精装修，你更偏爱哪一种?
---
并不是所有发行版都能够开箱即用，甚至面板和X都不会内置。又或者我们会受存储容量、使用习惯等因素影响而产生强烈的意愿自行搭配软件生态。本文长线更新记录了个人日常使用习惯中的一些实用命令和小细节作为备忘，而这也是建立技术博客的初衷。  

## 外观类
### Wallpaper Engine的Linux替代
安装VLC后执行`cvlc --video-wallpaper --no-audio /path/to/video.mp4`  
我可以不用，但不能没有  

## 文件处理类  
### 压缩解压  
仅以存储级别打包一个文件，在命令行键入  
`zip -r -0 folder.zip folder/`  
对于linux常见的压缩格式:  
`tar -zxvf xx.tar.gz`  
`tar -jxvf xx.tar.bz2`  
`xz -d && tar -xf`  

## 安装与卸载
### 清除所有已删除包的残馀配置文件  
```shell
apt autoremove --purge -y && dpkg -l |grep ^rc|awk '{print $2}' |sudo xargs dpkg -P
```
适合强迫症患者  

## 硬件调试  
### 蓝牙扫描   
`fang -s`  
类似于aircrack-ng扫描WLAN设备，但不是所有的蓝牙硬件都支持长时间高频扫描  
### shell全局代理    
`export ALL_proxy=http://ip:port`  
并不总是有条件享受透明代理环境的，不是吗  
### 串口接驳
`picocom -b 波特率 /dev/ttyUSBx`  

## 性能  
### chrome硬件解码  
谷歌移除了91版本后Linux平台Hardware-accelerated video decode高级选项. 对于较新的Intel核显设备, 需要用`intel-media-va-driver-non-free`替代开源驱动, 并在命令行键入  
`sed -i 's/google-chrome-stable/google-chrome-stable --use-gl=desktop --enable-features=VaapiVideoDecoder' /usr/share/applications/google-chrome.desk`  

## 容器
### Podman实用镜像
HomeAssistant是社区生态最完善的智能家居解决方案之一  
```shell
podman create -it --name homeassistant --platform linux/arm64 --restart=unless-stopped --label io.containers.autoupdate=registry --net host -v ~/appdata/ha:/config:z -v /run/dbus:/run/dbus:ro -e TZ=Asia/Shanghai docker.io/homeassistant/home-assistant:stable
podman generate systemd --new --name homeassistant > /etc/systemd/system/container-homeassistant.service && systemctl daemon-reload && systemctl enable --now container-homeassistant.service
```

mosquitto是轻量级mqtt服务器的首选之一  
```shell
mosquitto_passwd -b /mosquitto/config/pwd_file username password
podman create -it --name mqtt --restart=unless-stopped --label io.containers.autoupdate=registry --privileged --net host -v ~/appdata/mqtt/data:/mosquitto/data:z -v ~/appdata/mqtt/conf:/mosquitto/config:z -v ~/appdata/mqtt/logs:/mosquitto/log:z  -e TZ=Asia/Shanghai docker.io/eclipse-mosquitto:latest
podman generate systemd --new --name mqtt > /etc/systemd/system/container-mqtt.service && systemctl daemon-reload && systemctl enable --now container-mqtt.service
```

xray是v2ray的超集，以抗检测代理协议方案多样性而知名
```shell
podman create -it --name xray --restart=unless-stopped --label io.containers.autoupdate=image --network=host --volume /etc/xray:/etc/xray:z docker.io/teddysun/xray:latest
podman generate systemd --new --name xray > /etc/systemd/system/container-xray.service && systemctl daemon-reload && systemctl enable --now container-xray.service
```

jumpserver是一款可用于生产环境的开源堡垒机，all in one镜像需要外挂数据库和redis  
```shell
podman create -it --name jms_all --restart=unless-stopped --label io.containers.autoupdate=registry --network host --privileged -e SECRET_KEY=your_secure_key -e BOOTSTRAP_TOKEN=your_token -e LOG_LEVEL=ERROR -e DB_HOST=127.0.0.1 -e DB_PASSWORD=123456 -e DB_NAME=jumpserver -e DB_USER=jumpserver -e DB_PORT=3306 -e REDIS_HOST=127.0.0.1 -e REDIS_PORT=6379 -e REDIS_PASSWORD=123456 -e DOMAINS=yourdomain.com:443 -v ~/appdata/jms_all/core/config:/opt/jumpserver/config:z -v ~/appdata/jms_all/koko/data:/opt/koko/data:z -v ~/appdata/jms_all/lion/data:/opt/lion/data:z -v ~/appdata/jms_all/magnus/data:/opt/magnus/data:z -v ~/appdata/jms_all/kael/data:/opt/kael/data:z -v ~/appdata/jms_all/chen/data:/opt/chen/data:z -v ~/appdata/jms_all/core/data:/opt/jumpserver/data:z docker.io/jumpserver/jms_all:latest
podman create -it --name mariadb --restart=unless-stopped --label io.containers.autoupdate=registry --network host -e MARIADB_ROOT_PASSWORD=123456 -v ~/appdata/mariadb/data:/var/lib/mysql:z -v ~/appdata/mariadb/conf:/etc/mysql/conf.d:z docker.io/library/mariadb:latest
podman create -it --name redis --restart=unless-stopped --label io.containers.autoupdate=registry --network host -v ~/appdata/redis/data:/data:z -v ~/appdata/redis/conf:/usr/local/etc/redis:z docker.io/library/redis:latest redis-server /usr/local/etc/redis/redis.conf --appendonly yes
podman generate systemd --new --name jms_all > /etc/systemd/system/container-jms_all.service && podman generate systemd --new --name mariadb > /etc/systemd/system/container-mariadb.service && podman generate systemd --new --name redis > /etc/systemd/system/container-redis.service && systemctl daemon-reload
```
