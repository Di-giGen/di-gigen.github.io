---
title: RouterOS安装sing-box容器实现透明代理
date: 2024-04-21 12:19:20
tags: [RouterOS,路由器,sing-box,容器,科学上网,透明代理,旁路由,Mikrotik]
category: [计算机技术]
lede: 科学上网系列(その二)
thumbnail: /img/ros-with-singbox/100516900.jpg
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
本文中容器镜像展开在路由器板载闪存根目录`/containers`；诸如日志、配置文件等持久化内容存储在`/containers/appdata`路径下，以便容器后续升级。  
```shell
container mounts add dst=/root/sing-box name=singbox_persist src=/containers/appdata/singbox
```

如果要设定容器的时区，可添加容器环境参数：  
```shell
/container/envs add key=TZ name=singbox_envs value=Asia/Shanghai
```

### 应用配置文件
作为本博客科学上网系列的续篇，本文将在客户端使用到[前篇](https://di-gigen.github.io/2024/11/21/caddy-sni-with-reality/)部署的xray代理节点，故模板中的`MY_SERVER_IP`、`MY_UUID`、`MY_PUBLIC_KEY`需要和服务端配置文件内容相匹配。  
<details>
<summary><font color="#E02222">/containers/appdata/singbox/config.json</font></summary>

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
                "address": "local",  //使用运营商分配DNS提高国内解析速度
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
                //解析节点域名
                "outbound": "any",
                "server": "resolver-dns"
            },
            {
                //DNS去广告(类adguard)
                "rule_set": ["geosite-category-ads-all"],
                "server": "block-dns"
            },
            {
                "type": "logical",
                "mode": "or",
                "rules": [
                    {"domain_suffix": ["msftconnecttest.com"]},  //修复FakeIP模式下Windows联网状态显示异常
                    {"rule_set": ["geosite-cn"]}
                ],
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
        "client_subnet": "NEIGHBOUR_IP",  //填入一个属地IP，让DNS就近解析接入点
        "fakeip": {
            "enabled": true,
            "inet4_range": "198.18.0.0/15"
        }
    },
    "inbounds": [
        {
            "tag": "tun-in",
            "type": "tun",
            "address": ["172.19.0.1/30"],
            "stack": "gvisor",  //用户协议栈效率更高
            "auto_route": true,
            "sniff": true,  //嗅探协议类型和域名
            "sniff_override_destination": true  //目标IP覆写为域名，在服务端进行DNS解析
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
            //节点优选(5分钟频度检测比较延迟)
            "tag": "auto",
            "type": "urltest",
            "url": "https://www.gstatic.com/generate_204",
            "interval": "5m",
            "interrupt_exist_connections": true,
            "outbounds": ["VPS节点01","VPS节点02"]
        },
        {
            //vless+vision+reality协议组合
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
            //...
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
                "rules": [{"inbound": "dns-in"},{"protocol": "dns"}],
                "outbound": "dns-out"
            },
            {
                "type": "logical",
                "mode": "or",
                "rules": [{"rule_set": ["geosite-category-ads-all"]}],
                "outbound": "block"
            },
            {
                //访问局域网地址、BT下载、访问中国大陆地址走直连
                "type": "logical",
                "mode": "or",
                "rules": [{"ip_is_private": true},{"protocol": ["bittorrent"]},{"rule_set": ["geoip-cn","geosite-cn",]}],
                "outbound": "direct"
            }
        ],
        "auto_detect_interface": true,
        "final": "Proxy"
    }
}
```
</details>

### 下载镜像   
设置镜像源并拉取容器镜像，若被屏蔽请自行寻找[替代镜像](https://ghcr.nju.edu.cn)或外挂代理，此处不过多赘述。如果配置文件正确，容器服务启动后可以在`http://10.0.0.2/ui`看到一个简洁的web管理面板。为预防容器内存溢出影响主路由，还应添加容器模块可调用内存上限(这里配置为板载内存的一半512MB)  
```shell
/container/config/set registry-url=https://ghcr.io tmpdir=/tmp ram-high=512.0MiB
/container/ add remote-image=sagernet/sing-box:latest cmd="run -D /root/sing-box -C /root/sing-box" comment=singbox interface=veth_singbox mounts=singbox_persist envlist=singbox_envs root-dir=/containers/singbox  start-on-boot=yes
```

### 容器内IPv4转发
首先通过`container/print`命令确认sing-box容器的编号(本例程中假定为0)，然后使用`container/shell 0`进入容器内shell环境开启内核转发：  
```shell
echo "1" > /proc/sys/net/ipv4/ip_forward
```

## 旁路由模式
### 添加FakeIP静态路由
按照上文配置启动的sing-box容器`10.0.0.2`实质成为了Mikrotik网关`10.0.0.1`的旁路由，只是物理层在同一路由器设备上。当终端向旁路发送地址请求并且命中代理路由规则，sing-box会返回`198.18.0.0/15`段上的一个伪造IP，并通过VPS获取目标实际IP地址开始传输数据。FakeIP节省了终端直接向VPS质询域名解析结果的过程并降低延迟。这需要配置防火墙允许指向FakeIP的数据包转发到sing-box的虚拟接口  
```shell
/ip/route add comment=fakeip disabled=no distance=1 dst-address=198.18.0.0/15 gateway=10.0.0.2 pref-src=0.0.0.0 routing-table=main suppress-hw-offload=yes
```

### 按需配置终端指向旁路由
旁路代理的优势在于不会对主路由的拓扑结构和正常上网产生影响，需要让特定设备使用旁路代理的方式有很多，并且在RouterOS上可以替它们完成配置并作为缺省下发。  
对于希望流量完全被旁路接管的情况，可以将网关指向旁路由，TCP和UDP数据都直接进入虚拟接口`tun-in`被代理软件处理。此配置可以在RouterOS上保存为一组命名为`side_gw`DHCP预设：  
```shell
/ip/dhcp-server/option add code=3 name=side_gw value="'10.0.0.2'"
```

以上设置的缺陷是如果代理服务发生故障，被旁路由接管的设备将完全无法上网。如果您希望被代理设备的路由也具备一定程度的自愈性，可以只修改设备主DNS服务器指向旁路由，而备用DNS服务器指向主路由。旁路由正常工作时，设备上网的DNS质询命中代理规则，后续的数据请求会发送到代理服务提供的FakeIP并被处理。代理服务因故障无法响应时，则DNS质询回落到主路由上，设备仍可访问未被屏蔽的网站。通过以下命令在RouterOS将DHCP预设保存为`leak_protect`  
```shell
/ip/dhcp-server/option add name=leak_protect code=6 value="'10.0.0.2''10.0.0.1'"
```

## 透明代理模式
尽管上述旁路由能通过细化定义路由规则满足绝大多数网络环境的代理需求，但仍存在不使用域名请求的网络应用无法被旁路由DNS服务探测并接管，解决这个问题需要使用透明代理将请求这些特殊的IP的数据包转发到sing-box容器`tun-in`接口。  
以telegram的服务器为例，通过抓包或查[geo表](https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/refs/heads/sing/geo/geoip/telegram.json)将需要代理的IP(段)登记到`proxy_cidr`IP组  
```shell
/ip/firewall address-list
add address=91.105.192.0/23 comment=telegram list=proxy_cidr
add address=91.108.4.0/22 comment=telegram list=proxy_cidr
add address=91.108.8.0/21 comment=telegram list=proxy_cidr
add address=91.108.16.0/21 comment=telegram list=proxy_cidr
add address=91.108.56.0/22 comment=telegram list=proxy_cidr
add address=95.161.64.0/20 comment=telegram list=proxy_cidr
add address=149.154.160.0/20 comment=telegram list=proxy_cidr
add address=185.76.151.0/24 comment=telegram list=proxy_cidr
```

并不是所有的终端都使用透明代理，假设您安装了Telegram App的智能手机IP地址为10.0.0.100，可以为这些设备建立一份透明代理服务白名单`vip_client`：  
```shell
/ip/firewall/address-list add address=10.0.0.100/32 comment=mobilephone list=vip_client
```

配置防火墙为请求`proxy_cidr`清单内地址的`vip_client`终端发送的数据包打标并为其创建转发至sing-box虚拟接口的路由规则  
```shell
/routing/table add disabled=no fib name=proxy_cidr
/ip/firewall/mangle add chain=prerouting action=mark-routing new-routing-mark=proxy_cidr passthrough=yes src-address-list=vip_client dst-address-list=proxy_cidr
/ip/route add comment=proxy_cidr disabled=no distance=1 dst-address=0.0.0.0/0 gateway=10.0.0.2 routing-table=proxy_cidr suppress-hw-offload=yes
```
通常情况下预装RouterOS出厂的Mikrotik设备会内置一些缺省防火墙规则，我们不会在本文中讨论与应用实现无关的安全配置，但如果您采用虚拟机等方式全新安装软路由，需要确保内网开启了转发。  
被转发到旁路由的tun后，Telegram应用服务器IP请求命中大陆白名单的回落规则成功被sing-box代理，补完了代理的完整性。  
