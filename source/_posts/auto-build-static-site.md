---
title: hexo建站自动化维护技巧
date: 2019-09-27 23:09:49
tags: [运维,hexo,建站]
category: [计算机技术]
thumbnail: /img/auto-build-static-site/91149756.jpg
lede: 为你的hexo+gitpage建站方案引入自动发布、平滑升级等特性  
---
hexo作为开发活跃且的静态框架受到广大博主和站长们的喜爱和二次开发, 如果手上有Freenom的域名并且通过gitpage方式部署的话, 长期运营甚至不会产生任何费用. 今年Freenom暗改了免费域名的申请政策之后, 薅羊毛变得十分困难, 且域名容易被指滥用而收回, 而github page相对服务器部署方式来说就没有宕机或欠费导致域名被闲置的风险.  
本文的服务对象为使用hexo框架+gitpage方案建站保域名的小站长, 并希望利用git自身特性进一步提高网站部署的自动化程度与可靠性. 此处不再赘述网络上广泛存在的hexo博客搭建部分教程.   

## 平滑升级  
网站的个性化定制在hexo框架上主要集中在主题配置上. 我们将主题集成到网站仓库时, 通常希望在不损失配置的情况下接收主题的更新(在后续步骤中, 此特性亦可保证在每一次生成生成网页都是拉取最新的源码). 为做到这一点, 首先需要用`git submodule add <submodule_url> themes/`命令将主题添加为项目的子模块  , 之后就可以在`.gitmodule`文件中找到所有引用的子模块URL与已经拉取的本地目录之间的映射了.  
为了避免在更新主题子模块的时候覆盖掉配置文件, 还需要将主题配置文件放到项目的主要目录下生效. 早期的hexo主题需要开发者主动引入[data files](https://hexo.io/docs/data-files.html)特性, 而在hexo版本5.0之后, 只需要将主题配置文件重命名为`_config.<theme_name>.yml`放置到项目根目录, 即可直接被hexo读取.   

## 自动发布  
hexo框架特性使然允许的开发环境是唯一的. 这意味着网站的发布环境只能部署在一台本地设备上, 几乎没有容灾能力.  
而通过持续集成来管理网站git项目的发布, 将生成的网页文件和hexo开发环境用2条分支(当然也可以是2个独立仓库, 但没必要)分别存放和管理. 这样做的好处是, 开发者可以在任何一台电脑上拉取开发分支(以下简称Source Branch)立刻开始工作, 再由持续集成服务自动将改动后的源码重新渲染成网页发布到gitpage所在的分支(以下简称Content Branch)上, 达到接近动态站一键发布的便利性.  

### CI宿主服务
现有的CI服务提供商当中同时支持Windows和Linux双开发环境的仅有[AppVeyor](https://www.appveyor.com/), 推荐非Github仓库使用.  
1. 创建hexo网站git代码仓库  
在上一节中创建的GitHub仓库中创建一条新的分支并设置为用于存放gitpage静态网页(在本例程中的名称为master).  
2. 登录AppVeyor并绑定Project  
在[登录页面](https://ci.appveyor.com/login)使用github账号注册后, 添加步骤1中创建的hexo网站git仓库到[CI项目](https://ci.appveyor.com/projects/)中.
3. 添加CI脚本到Source分支  
AppVeyor的CI脚本文件为`appveyor.yml`, 在Source分支的根目录下新建此文件并添加下面的内容:  

```yaml
clone_depth: 5

# 限制只编译source branch
branches:
  only:
  - <source_branch_name>

environment:
  my_variable:
    secure: <github_access_token>

install:
  - git submodule update --init --recursive # 每一次编译之前拉取最新的主题submodule
  - node --version
  - npm --version
  - npm install
  - npm install hexo-cli -g

build_script:
  - hexo generate

artifacts:
  - path: public

on_success:
  - git config --global credential.helper store
  - ps: Add-Content "$env:USERPROFILE\.git-credentials" "https://$($env:my_variable):x-oauth-basic@github.com`n"
  - git config --global user.email "%GIT_USER_EMAIL%"
  - git config --global user.name "%GIT_USER_NAME%"
  - git clone --depth 5 -q --branch=%TARGET_BRANCH% %STATIC_SITE_REPO% %TEMP%\static-site
  - cd %TEMP%\static-site
  - del * /f /q
  - for /d %%p IN (*) do rmdir "%%p" /s /q
  - SETLOCAL EnableDelayedExpansion & robocopy "%APPVEYOR_BUILD_FOLDER%\public" "%TEMP%\static-site" /e & IF !ERRORLEVEL! EQU 1 (exit 0) ELSE (IF !ERRORLEVEL! EQU 3 (exit 0) ELSE (exit 1))
  - git add -A
  - git commit -m "Update Static Site"
  - git push origin %TARGET_BRANCH%
  - appveyor AddMessage "Static Site Updated"
```

其中涉及到的参数`<github_access_token>`需要自行在github设置中生成, 遵循[官方步骤](https://ci.appveyor.com/tools/encrypt)将生成后的明文token[加密](https://ci.appveyor.com/tools/encrypt)后粘贴到脚本中保存.  

4. 在AppVeyor Portal中设置CI脚本中的变量  

### 更好的方式: Github Actions  
如果你追求的是在Github一站式解决自动发布问题而不是注册额外的服务, 那么可以考虑Action服务, Hexo[官方支持](https://hexo.io/zh-cn/docs/github-pages)将公共或私密(收费)仓库source以artifact形式发布.  
