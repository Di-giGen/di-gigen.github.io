---
title: 与Caddy复用端口的Hysteria2代理方案
date: 2024-11-30 19:58:00
tags: [网络]
category: [计算机技术]
lede: 科学上网系列(その三)
thumbnail: /img/caddy-with-hysteria2/tcp&udp-visualized.jpeg
---
## 前言
借助xtls相关协议降低tls流量特征、利用伪装站tcp转发以隐藏VPS地址等技术手段，我们在系列[前篇](https://di-gigen.github.io/2024/11/21/caddy-sni-with-reality/)已经搭建了基于TCP的高隐匿性代理服务。而对于部分侧重发包而非传输可靠性的使用场景如流媒体、直播、游戏联机的场景，如果您的VPS线路优质，那么自建基于UDP协议的Hysteria2节点能帮您省下游戏加速器的花费。本文将为使用Caddy建站的服主们提供Hysteria2的端口复用伪装部署方式。  

## 先决条件 
本文假定用户仍选择本系列推荐的Caddy+sing-box软件组合，在服务器上架设博客和代理节点。  
依然建议保留前篇的iptables的转发配置，降低遭受服务提权攻击的风险  

## 预期效果
* Caddy自动申请通配符证书并接管443端口转发来的TCP流量，处理网页请求反代至博客及服务器上其他网络服务  
* sing-box接管443端口转发来的UDP流量，处理Hysteria2代理流量的同时，也响应对HTTP/3的请求，提供博客域名的QUIC访问方式  

## 安装与配置
### Caddy
为保障接口复用，Caddyfile的重点在于关闭Caddy自身的HTTP/3支持以避免占用UDP 443端口  
```
{
    https_port 8443
    servers :8443 {
        protocols h1 h2 h2c
    }
    log {
        output file /log/path/error.log
        format console
        level error
    }
    admin off
    acme_dns cloudflare MY_API_TOKEN
    email my@email.com
    layer4 {...}
}
*.mydomain.com {
    encode gzip zstd
    log {
        output file /log/path/access.log
        format console
    }
    @sub host sub.mydomain.com
    handle @sub {
        reverse_proxy 127.0.0.1:8080

        ## other services
        ...
    }
    
    ## Fallback for otherwise unhandled domains
    handle {
        abort
    }
}
```
Caddy的安装不再赘述。本方案无需使用[前篇](https://di-gigen.github.io/2024/11/21/caddy-sni-with-reality/)的L4路由功能，但我们仍建议保留此插件，因为sing-box能很方便地同时接管多种的代理协议，这样您可以针对不同使用需求灵活选用reality还是hysteria2。  

### sing-box
sing-box将在`7894`监听hysteria2的代理请求,强制命令客户端使用BBR拥塞控制算法。当443端口遭受主动检测的时候，会返回通过HTTP/3访问主站`sub.mydomain.com`的网页以达到伪装目的。  
除了需要补全`MY_USERNAME`和`MY_PASSWORD`，您还需要根据服务器文件结构酌情调整`key_path`和`certificate_path`，通常无根Caddy自签的证书会被存储在`~/.local/share/caddy/certificates/`，模板中展示的是采用前篇Podman方式部署sing-box时Caddy自签证书路径在容器内的映射。  
```json
{
    "log": {
        "disabled": false,
        "level": "error",
        "output": "/your/log/path/box.log",
        "timestamp": true
    },
    "inbounds": [{
        "tag": "hy2-in",
        "type": "hysteria2",
        "listen": "::",
        "listen_port": 7894,
        "ignore_client_bandwidth": true,
        "masquerade": "http://localhost:8080",
        "users": [{
                    "name": "MY_USERNAME",
                    "password": "MY_PASSWORD"
        }],
        "tls": {
            "enabled": true,
            "server_name": "sub.mydomain.com",
            "key_path": "/etc/ssl/private/acme-v02.api.letsencrypt.org-directory/wildcard_.mydomain.com/wildcard_.mydomain.com.key",
            "certificate_path": "/etc/ssl/private/acme-v02.api.letsencrypt.org-directory/wildcard_.mydomain.com/wildcard_.mydomain.com.crt"
        }
    }],
    "outbounds": [
        {
            "tag": "direct",
            "type": "direct"
        },
        {
            "tag": "block",
            "type": "block"
        }
    ],
    "route": {
        "rule_set": [
            {
                "type": "remote",
                "tag": "geoip-cn",
                "format": "binary",
                "url": "https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/sing/geo/geoip/cn.srs",
                "download_detour": "direct"
            },
            {
                "type": "remote",
                "tag": "geosite-cn",
                "format": "binary",
                "url": "https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/sing/geo/geosite/cn.srs",
                "download_detour": "direct"
            }
        ],
        "rules": [{
            "type": "logical",
            "mode": "or",
            "rules": [{"rule_set": ["geoip-cn","geosite-cn"]},{"ip_is_private": true},{"protocol": ["bittorrent"]}],
            "invert": false,
            "outbound": "block"
        }],
        "final": "direct"
    }
}
```
### 性能调优
调增UDP缓冲区 (此处为16M)  
```shell
echo "net.core.rmem_max=16777216" >> /etc/sysctl.conf && echo "net.core.wmem_max=16777216" >> /etc/sysctl.conf
```

### 防火墙适配
如果没有其它需求，可以直接复用[前篇](https://di-gigen.github.io/2024/11/21/caddy-sni-with-reality/)的防火墙配置文件，唯一的区别在于UDP 8443改为了sing-box监听。  

### 客户端配置
补全服务器配置文件中对应参数后，即在各平台客户端中使用此分享链接：  
```
hysteria2://MY_PASSWORD@sub.mydomain.com:443?insecure=0
```

## 完成！
启动Caddy与sing-box服务。此时可通过`sudo ss -lnp |grep :8443`命令查询接口占用情况。若配置无误，可以看到Caddy和sing-box分别监听8443端口的TCP和UDP。  
如果有余力，还可以模拟网络審查官员使用`curl --http3-only https://sub.mydomain.com`对443端口进行扫描，将会正常返回博客首页的内容。  
