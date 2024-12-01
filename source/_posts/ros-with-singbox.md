---
title: RouterOS安装sing-box容器实现透明代理
date: 2024-07-03 11:05:20
tags: [网络,路由器]
category: [计算机技术]
lede: 科学上网系列(その二)
thumbnail: /img/ros-with-singbox/100516900.png
---

## 前言  
RouterOS作为Mikrotik原厂软路由系统，在路由器的本职工作上往往展现出远超OpenWrt的可靠性。本文旨在充分利用ROS 7.8+版本引入的[container](https://help.mikrotik.com/docs/display/ROS/Container)功能在一些CPU算力盈余的Mikrotik机型(以RB5009为例)上部署sing-box实现广义层面的透明代理(即路由器为局域网设备统一提供代理服务)。同时充分发挥ROS的配置灵活性，将对主路由正常上网的影响降至最低。  

## 容器配置
### 网络  
在本文中Mikrotik路由器提供了局域网段`10.0.0.1/24`，其中sing-box容器被分配并固定为`10.0.0.2`  
```shell
/interface/veth/add name=veth_singbox address-10.0.0.2 gateway=10.0.0.1
/interface/bridge/port add bridge=bridge interface=veth_singbox
```

### 文件系统
本文中容器镜像展开在路由器板载闪存根目录`/containers`；诸如日志、配置文件等持久化内容存储在`/containers/appdata`路径下。  
```shell
container mounts add dst=/root/sing-box name=singbox_persist src=/containers/appdata/singbox
```

### 应用配置文件
作为本博客科学上网系列的续篇，本文将在客户端使用到[前篇](https://di-gigen.github.io/2024/11/21/caddy-sni-with-reality/)部署的xray代理节点，故模板中的`MY_SERVER_IP`、`MY_UUID`、`MY_PUBLIC_KEY`需要和服务端配置文件内容相匹配。客户端路由规则为最简单的本地直连+大陆白名单+广告屏蔽，支持多节点自动切换(需补充VPS节点02或者添加更多)，同时在`http://10.0.0.2/ui`提供简易web管理面板。  
```json
{
    "experimental": {
        "cache_file": {
            "enabled": true,
            "path": "cache.db",
            "store_fakeip": true
        },
        "clash_api": {
            "external_ui": "ui",
            "external_controller": "0.0.0.0:80",
            "external_ui_download_detour": "Proxy",
            "default_mode": "rule"
        }
    },
    "log": {
        "disabled": false,
        "level": "error",
        "timestamp": true
    },
    "dns": {
        "servers": [
            {
                "tag": "remote-dns",
                "address": "https://one.one.one.one/dns-query",
                "address_resolver": "resolver-dns",
                "detour": "Proxy"
            },
            {
                "tag": "resolver-dns",
                "address": "223.5.5.5",
                "detour": "direct"
            },
            {
                "tag": "fakeip-dns",
                "address": "fakeip"
            },
            {
                "tag": "block-dns",
                "address": "rcode://success"
            }
        ],
        "rules": [
            {
                "rule_set": ["geosite-category-ads-all"],
                "server": "block-dns"
            },
            {
                "outbound": "any",
                "server": "resolver-dns"
            },
            {
                "rule_set": ["geosite-cn"],
                "server": "resolver-dns"
            },
            {
                "query_type": ["A"],
                "rewrite_ttl": 1,
                "server": "fakeip-dns"
            }
        ],
        "final": "remote-dns",
        "strategy": "ipv4_only",
        "fakeip": {
            "enabled": true,
            "inet4_range": "198.18.0.0/15"
        }
    },
    "inbounds": [
        {
            "tag": "tun-in",
            "type": "tun",
            "inet4_address": ["172.19.0.1/30"],
            "stack": "system",
            "sniff": true,
            "auto_route": true,
            "sniff_override_destination": true,
            "gso": true
        },
        {
            "tag": "dns-in",
            "type": "direct",
            "listen": "::",
            "listen_port": 53,
            "sniff": true
        }
    ],
    "outbounds": [
        {
            "tag": "Proxy",
            "outbounds": ["auto","VPS节点01","VPS节点02","direct"],
            "default": "auto",
            "type": "selector",
            "interrupt_exist_connections": true
        },
        {
            "tag": "direct",
            "type": "direct"
        },
        {
            "tag": "dns-out",
            "type": "dns"
        },
        {
            "tag": "block",
            "type": "block"
        },
        {
            "tag": "auto",
            "type": "urltest",
            "url": "https://www.gstatic.com/generate_204",
            "interval": "5m",
            "interrupt_exist_connections": true,
            "outbounds": ["VPS节点01","VPS节点02"]
        },
        {
            "tag": "VPS节点01",
            "type": "vless",
            "server": "MY_SERVER_IP",
            "server_port": 443,
            "uuid": "MY_UUID",
            "flow": "xtls-rprx-vision",
            "tls": {
                "enabled": true,
                "server_name": "itunes.apple.com",
                "utls": {
                    "enabled": true,
                    "fingerprint": "chrome"
                },
                "reality": {
                    "enabled": true,
                    "public_key": "MY_PUBLIC_KEY"
                }
            }
        },
        {
            "tag": "VPS节点02",
            ...
        }
    ],
    "route": {
        "rule_set": [
            {
                "tag": "geoip-cn",
                "type": "remote",
                "format": "binary",
                "download_detour": "direct",
                "update_interval": "1d",
                "url": "https://testingcf.jsdelivr.net/gh/MetaCubeX/meta-rules-dat@sing/geo/geoip/cn.srs"
            },
            {
                "tag": "geosite-cn",
                "type": "remote",
                "format": "binary",
                "download_detour": "direct",
                "update_interval": "1d",
                "url": "https://testingcf.jsdelivr.net/gh/MetaCubeX/meta-rules-dat@sing/geo/geosite/cn.srs"
            },
            {
                "type": "remote",
                "format": "binary",
                "download_detour": "direct",
                "tag": "geosite-category-ads-all",
                "update_interval": "1d",
                "url": "https://testingcf.jsdelivr.net/gh/MetaCubeX/meta-rules-dat@sing/geo/geosite/category-ads-all.srs"
            }
        ],
        "rules": [
            {
                "type": "logical",
                "mode": "or",
                "rules": [{"protocol": "dns"}],
                "outbound": "dns-out"
            },
            {
                "type": "logical",
                "mode": "or",
                "rules": [{"rule_set": ["geosite-category-ads-all"]}],
                "outbound": "block"
            },
            {
                "type": "logical",
                "mode": "or",
                "rules": [{"rule_set": ["geoip-cn","geosite-cn",]},{"ip_is_private": true}],
                "outbound": "direct"
            }
        ],
        "auto_detect_interface": true,
        "final": "Proxy"
    }
}                                                                                  
```
请将配置保存为`/containers/appdata/singbox/config.json`

### 下载镜像   
设置镜像源并下载，若被屏蔽请自行寻找[替代镜像](https://ghcr.nju.edu.cn)或通过代理下载，此处不过多赘述。  
```shell
/container/config/set registry-url=https://ghcr.io tmpdir=/tmp
/container/ add cmd="run -D /root/sing-box -C /root/sing-box" comment=singbox interface=veth_singbox mounts=singbox_persist root-dir=/containers/singbox  start-on-boot=yes
```

### 容器内IPv4转发
首先通过`container/print`命令确认sing-box容器的编号(本例程中假定为0)，然后使用`container/shell 0`进入容器内shell环境开启内核转发：  
```shell
echo "1" > /proc/sys/net/ipv4/ip_forward
```

## 旁路由设置
### 添加fakeip静态路由
让数据包可以通过sing-box的`tun-in`接口转发  
```shell
/ip/route add comment=fakeip disabled=no distance=1 dst-address=198.18.0.0/15 gateway=10.0.0.2 pref-src=0.0.0.0 routing-table=main suppress-hw-offload=yes
```

配置启动sing-box进程后，容器(10.0.0.2)实质成为了Mikrotik(网关10.0.0.1)的旁路由，只是物理层在同一路由器设备上。当终端向其发送地址请求一旦命中代理路由规则，sing-box会返回`198.18.0.0/15`段上的一个伪造IP用以拦截，并根据洁净DNS查询结果从目标地址请求数据并转发到终端。

### 配置终端DNS指向旁路由
以上部署不会对主路由处理的一般上网行为造成影响，而要让特定的设备终端使用旁路代理，仅需将DNS服务器地址指向sing-box容器IP地址。此步骤原本需要在终端上逐一配置，但RouterOS的DHCP服务支持统一设置分别下发。仅需使用以下命令录入一次，就可以通过Mikrotik路由器Web后台轻松为任意终端设备配置`leak_protect`DNS首选项，使其缺省DNS自动指向`10.0.0.2`。  
```shell
/ip dhcp-server option add name=leak_protect code=6 value="'10.0.0.2'"
```

## 透明代理设置
尽管上述旁路代理能通过细化定义路由规则满足绝大多数网络环境的代理需求，但仍存在不使用域名请求的网络应用无法被旁路由接管，为此需要针对这些特殊的IP请求地址配置指向sing-box容器`tun-in`接口的路由表来实现透明代理。  
以telegram的服务器为例，通过抓包或查[geo表](https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/refs/heads/sing/geo/geoip/telegram.json)将需要代理的IP(段)登记到`proxy_cidr`IP组  
```shell
/ip firewall address-list
add address=91.105.192.0/23 comment=telegram list=proxy_cidr
add address=91.108.4.0/22 comment=telegram list=proxy_cidr
add address=91.108.8.0/21 comment=telegram list=proxy_cidr
add address=91.108.16.0/21 comment=telegram list=proxy_cidr
add address=91.108.56.0/22 comment=telegram list=proxy_cidr
add address=95.161.64.0/20 comment=telegram list=proxy_cidr
add address=149.154.160.0/20 comment=telegram list=proxy_cidr
add address=185.76.151.0/24 comment=telegram list=proxy_cidr
```

用防火墙为请求`proxy_cidr`清单内IP的数据包打标并允许转发  
```shell
/ip/firewall add action=accept chain=forward comment=proxy_cidr dst-address-list=proxy_cidr
/ip/firewall/mangle add chain=prerouting action=mark-routing new-routing-mark=proxy_cidr passthrough=yes dst-address-list=proxy_cidr
```

添加路由表将打标数据包发送到sing-box容器的虚拟接口`veth_singbox`  
```shell
/routing/table add disabled=no fib name=proxy_cidr
/ip/route add comment=proxy_cidr disabled=no distance=1 dst-address=0.0.0.0/0 gateway=10.0.0.2 routing-table=proxy_cidr scope=30 suppress-hw-offload=yes target-scope=10
```

被转发到旁路由的tun后，Telegram应用服务器IP请求命中大陆白名单的回落规则成功被sing-box代理，补完了旁路由透明代理的完整性。  