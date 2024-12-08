---
title: Caddy实现单点登录认证
date: 2024-07-03 11:05:20
tags: [2FA,sso,caddy]
category: [计算机技术]
lede: 轻量Web服务sso解决方案
thumbnail: /img/sso-by-caddy/121154966.jpg
---
## 前言
在远程访问Homelab的问题上，隧道技术是被广泛应用于内网穿透的安全之选。而对于保密性要求较低且依赖持续性连接的服务(比如智能家居、私有化IM聊天语音服务等)，我们还是需要提供广域网可达的持久化地址。如果把家宽作为互联网服务出口，需要将动态公网IP通过DDNS解析为固定的域名，并申请SSL证书对传输进行加密。  
配置内网服务反代的时候我们往往会发现一些口令策略强度难以承受暴力破解或者根本不具备登录认证功能的资产(比如家用路由器的web后台，尽管我们不建议用户将此类设备发布在网上)，我们需要为其前置一个安全可靠的登录界面，即3A(认证Authentication、授权Authorization、账号Account)管理系统。  
由Homelab通常不会产生很大的访问量，我们可以选用Caddy作为Web Server，并集成插件来实现证书签发和单点登录，这比起分别部署nginx+acme+authelia三个服务要更简单。  

## 实操步骤
### 构建Caddy二进制文件  
尽管我们现在已经可以马上用一个最简单的配置文件让Caddy运行起来，但各发行版收录的安装包仅包含基础功能，并不能满足我们的需要：  
* 出于安全访问的需求，Caddy在收到访问内网服务请求时需要重定向到单点登录平台完成鉴权  
* 宽带的ISP通常会通过封锁常见端口，这也阻止了http和tls-alpn验证方式获取证书，为绕开限制需要使用DNS服务商的dns-challenge插件。这种验证方式也允许您申请通配符证书来接管多个子域名  
在这个例子中准备的域名由科赋锐提供解析服务  

从官方[包生成器](https://caddyserver.com/download?package=github.com%2Fcaddy-dns%2Fcloudflare&package=github.com%2Fgreenpau%2Fcaddy-security)，勾选需要的`cloudflare`和`caddy-security`插件，以及运行caddy的设备对应的架构(本例中为amd64)，直接下载到集成好的版本，并赋予执行权限。  

### 为caddy配置通配符证书签发、反代参数和多因子认证  
如前文所述，在可预期的家宽端口限制条件下，我们假定Caddy在互联网出口IP的高位端口`8443`监听访问请求至对应的两个Homelab服务`sub1`和`sub2`，这也意味着所有家用域名需要补全此端口，例如3A管理后台地址应设为`auth.mydomain.com:8443`。由于涉及多个子域，Caddy为其统一申请通配符证书以简化配置。作为对照，`sub1`接入单点登录`sub2`保持裸连。  
<details>
<summary><font color="#E02222">Caddyfile</font></summary>

```
{
    https_port 8443
    acme_dns cloudflare MY_API_KEY
    email my@email.com
    log {
        level error
        output file /log/path/error.log
    }
    admin off
    order authenticate before respond
    order authorize before basicauth
    security {
        local identity store localdb {
            realm local
            path /etc/caddy/users.json
        }
        authentication portal myportal {
            crypto default token lifetime 3600
            crypto key sign-verify {env.JWT_SHARED_KEY}
            enable identity store localdb
            cookie domain mydomain.com
            ui {
                links {
                    "Server 1" https://sub1.mydomain.com:8443 icon "las la-server"
                    "Server 2" https://sub2.mydomain.com:8443 icon "las la-server"
                    "Portal Settings" /settings icon "las la-cog"
                }
            }
            transform user {
                match realm local
                require mfa
                action add role authp/user
            }
        }
        authorization policy users_policy {
            set auth url https://auth.mydomain.com:8443/
            allow roles authp/admin authp/user
            crypto key verify {env.JWT_SHARED_KEY}
            acl rule {
                comment allow users
                match role authp/user
                allow stop log info
            }
            acl rule {
                comment default deny
                match any
                deny log warn
            }
        }
    }
}
*.mydomain.com {
    encode gzip zstd
    log {
        output file /etc/caddy/access.log 
        format console
    }
    ## SSO backend
    @auth host auth.mydomain.com
    handle @auth {
        authenticate with myportal
    }
    ## 内网服务1
    @sub1 host sub1.mydomain.com
    handle @sub1 {
        route /path* {
            authorize with users_policy
            reverse_proxy IP:PORT
        }
        route {
            reverse_proxy IP:PORT
        }
    }
    ## 内网服务2
    @sub2 host sub2.mydomain.com
        handle @sub2 {
            reverse_proxy IP:PORT
        }
    ## Fallback for otherwise unhandled domains
    handle {
        abort
    }
}
```
</details>

依照模板配置后，请求`sub1.mydomain.com:8443/path`内容将被重定向至单点登录界面完成鉴权后才可访问而`sub1.mydomain.com:8443`根路径和整个`sub2`不受此限制。  
