---
title: 米家扫地机器人刷机实战
date: 2019-06-23 22:32:19
tags: [智能家居,homeassistant,刷机]
category: [消费电子]
lede: 国行特供受害者笔记
thumbnail: /img/free-your-mi-vacuum-from-cloud/99192422.jpg
---
## 前言  
本文采用一种不同于[Miot](https://github.com/al-one/hass-xiaomi-miot)插件的方式将Roborock S50米家扫地机器人接入Home Assistant，在保留绝大部分原厂功能的同时停止向米家服务器的数据传输。  

## 前期准备: 获取米家设备token  
对于安卓用户，下载修改版的[米家App](https://cloud.mail.ru/public/33J3/t817u4Snr)按照常规方式进行配对连接后，在应用程式里选中扫地机器人，进入菜单配置/常规设置/网络信息选项卡查看当前token(已过时)
对于PC用户，可以遵循[xiaomi_miio集成步骤](https://www.home-assistant.io/components/vacuum.xiaomi_miio/#retrieving-the-access-token)跳过常规方式配对连接直接获取token(更安全)  

## 原厂固件转区(可选)
非必要。但考虑到消费电子产品在大陆地区与其他地区的用户使用条款之间的巨大差异，即使不进行进一步的Hack，刷写国际版固件的设备也将使用户享受到诸如[GDPR](https://eugdpr.org/)等政策的保护，推荐遵循[这篇文章](https://xiaomirobot.wordpress.com/transformer-un-modele-china-edition-vers-international/)执行转区操作，并备份[固件](https://cdn.awsbj0.fds.api.mi-img.com/rubys/updpkg/v11_001886.fullos.pkg)。  

## 刷入Valetudo
本文二编时，这款笔者2018年开始使用的三方固件已经形成了优质且操作简便的[文档](https://valetudo.cloud/pages/general/getting-started.html#installing_valetudo)，原实操记录已失去参考意义。读者应对照项目支持型号，在社区确认国际型号与国行特供版本之间的刷写互通性再执行操作，以免令扫地机器人变砖并失去售后。  

## 语音客制化
如果想让扫地机器人语音更换为特定国家，可以下载[语音包](https://dustbuilder.xvm.mit.edu/pkg/voice/)并通过此命令刷写  
```shell
mirobo --ip [vacuum_ip] --token [your_token] install-sound /path/to/xx.pkg
```

## 追记
本文的初衷是用自由软件驱动优质商业硬件，在享受产品功能的同时尽可能保障个人数据隐私。米家生态的发展历程，开源社区见证了太多的封闭政策收紧、在线服务跑路、推陈出新背后的降本降级等问题疲于跟进，反映在普通消费者身上则是无力填补与企业势的信息差和技术资本鸿沟，每一次智能产品的购买都缺乏安全感。而当我在市场上已经找不到可以被Hack的硬件时，也许应该换个思路，对智能家居设备的可达网域和出站数据进行限制？    
