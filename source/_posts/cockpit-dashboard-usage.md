---
title: Linux服务器面板推荐：Cockpit
date: 2023-11-23 16:47:04
tags: [运维,VPS,面板,caddy,2FA]
category: [计算机技术]
thumbnail: /img/cockpit-dashboard-usage/88238738.jpg
lede: 开源可信的服务器管理面板  
---
## 前言
在tls伪装为主流代理方案的时代，一个典型的v2ray构建通常是WebSocket协议+TLS传输+Web站点伪装(特殊情况下还会给他们套上一层CDN抵抗针对源IP的封锁)，很多用户选择使用一个Web服务器管理面板一举两得地作为伪装站点内容并提供管理服务器的数据库、文件、性能监控等实用功能。但我相信还是有不少用户抵触诸如[1Panel](https://1panel.cn/)、[宝塔面板](https://www.bt.cn/)之类缺乏中立性背景或存在后门隐患的面板。在此推荐由红帽公司推出并用于自家商业发行版RHEL的[Cockpit](https://cockpit-project.org/)。  

## 安装与配置
Cockpit已经被[收录](https://cockpit-project.org/running.html)在常见发行版的包管理器中，例如debian-based发行版就可以直接用命令`sudo apt install cockpit`完成安装。

### 可选功能组件
#### 容器管理
除了cockpit本体，为了管理容器我们还需要安装其功能插件`cockpit-podman`(当然，服务器也必须安装有podman)  
Cockpit在理念上与红帽公司本家的发行版一脉相承，一般用户感知最强的容器管理服务就早早地从docker转向了podman。但是这种转变对于容器的部署与管理并没有影响，Podman完全兼容docker容器镜像且二者在指令语法上的几乎完全相同。podman具备允许用户以无根方式部署低权限容器服务、无需依赖daemon守护进程、支持一键无痛更新容器且支持服务化自动更新而无需watchtower等优点。  

#### 2FA认证
默认状态下Cockpit直接通过服务器Linux系统的用户名密码进行登录，但这对于暴力破解广泛存在的公网环境是十分危险的。所幸其支持Linux原生PAM身份认证框架，因此也可以使用`libpam`的2FA令牌认证插件。在本文中我们将以`google-authenticator`为例，在debian-based发行版中，它的包名是`libpam-google-authenticator`。  
完成安装后使用`google-authenticator -t -d -f -r 3 -R 30 -W`命令生成一个带有密钥的二维码，在智能手机上安装诸如[google authentificator](https://play.google.com/store/apps/details?id=com.google.android.apps.authenticator2)之类的令牌管理器扫描此二维码，通过向导它会要求您输入密钥(或者刷新二维码重试)以完成令牌激活。之后您可以找到记载了该令牌紧急恢复码的文本文件`~/.google_authenticator`，请妥善保管。  
接下来我们需要使用以下命令改变Cockpit的登录策略为必须使用2FA令牌：  
```shell
`echo "auth required pam_google_authenticator.so" >> /etc/pam.d/cockpit && systemctl restart cockpit.service`
```

### 修改访问链接
正常情况下，Cockpit服务会运行在`9090`端口上，我们需要使用SSL加密访问面板的通信避免敏感数据泄露，这通常由http server配合acme证书申请服务来完成，本文中我们使用Caddy签发证书和反向代理。有关此反代工具的介绍和安装不再赘述，请参考[官方文档](https://caddyserver.com/docs/install)完成安装。  
为了体现Cockpit的灵活性，我们假定在服务器上已经存在一个动态站服务存在于`8080`端口的情况下，使面板与网站共用同一个域名`sub.mydomain.com`。
首先新建一个面板配置文件声明site_url：  
<details>
<summary><font color="#E02222">/etc/cockpit/cockpit.conf</font></summary>

```conf
[WebService]
Origins = https://sub.mydomain.com wss://sub.mydomain.com
ProtocolHeader = X-Forwarded-Proto
UrlRoot=/mgmt
```
</details>

然后编辑Caddy与面板配置相对应：  

<details>
<summary><font color="#E02222">/etc/caddy/Caddyfile</font></summary>

```
{
    https_port 443
    log {
        output file /log/path/error.log
        format console
        level error
    }
    admin off
    acme_dns cloudflare MY_API_TOKEN
    email my@email.com
}
*.mydomain.com {
    encode gzip zstd
    log {
        output file /log/path/access.log
        format console
    }
    @sub host sub.mydomain.com
    handle @sub {
        ## website
        reverse_proxy 127.0.0.1:8080   
        ## cockpit
        reverse_proxy /mgmt/* 127.0.0.1:9090 {
            transport http {
                tls_insecure_skip_verify
            }
        }
        rewrite /mgmt /mgmt/
    }    
    ## Fallback for otherwise unhandled domains
    handle {
        abort
    }
}
```
</details>

补全证书Challenge配置项`MY_API_TOKEN`和`my@email.com`、创建日志文件`/log/path/error.log`和`/log/path/access.log`后重启服务，此时`https://sub.mydomain.com/mgmt`即为Cockpit管理面板的访问链接。  
