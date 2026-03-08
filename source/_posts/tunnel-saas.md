---
title: Tunnel如何优选
date: 2026-03-08 13:38:00
tags: Cloudflare
categories: 教程
---
如果你在**Cloudflare**网页直接创建隧道，并使用自定义主机名的话  
你的网站会出现神秘的404问题  
当时我也没有搞懂，无奈之下，做了张梗图：  
<img src="/img/tunnel-meme.jpg" width="30%" height="30%">
~~(我是云服务文盲)~~
  
偶然之间我找到了一篇博文 [使用CloudFlare SaaS的同时使用Cloudflare Tunnel](https://blog.cosmiccat.net/2024/01/637/)  
我跟着操作，果然解决了神秘的404问题  
但是这篇博文，他**被墙了**  
再加上里面放了很多tunnel的官方文档的链接，还得自己去翻，很容易掉坑里  
为了方便大家，我把核心步骤复述并简化如下，省去翻官方文档的麻烦。

---

## ❓ 为什么会出现 404？

Cloudflare 的 **云端管理型隧道**（Cloudflare Dashboard 创建的）有一个限制：  
**只能绑定已托管在 Cloudflare DNS 下的域名**。  
如果你用的是第三方 DNS（比如阿里云、GoDaddy），或者通过 CNAME 接入 Cloudflare SaaS（如优选 IP 服务），  
那么 Tunnel 根本不会识别这些“外部域名”，直接返回 404。

---
# 解决方案
本地管理型隧道可以支持绑定任意域名。于是解决方法为使用本地管理型隧道而不是官方默认推荐使用的云端管理型。  

**注意！以下内容需要你注册Cloudflare并开通Zero Trust**  
## 安装cloudflared  
前往[Cloudflared Github Releases页面](https://github.com/cloudflare/cloudflared/releases/latest)获取最新版cloudflared安装包
## 登录Cloudflare  
```` bash
cloudflared login
````
## 创建本地自托管Tunnel
```` bash
cloudflared tunnel create <NAME>
````
## 创建配置文件
```` bash
vim ~/.cloudflared/config.yml
````
<br>

```` yml ~/.cloudflared/config.yml
tunnel: 上一步中提示的uuid
credentials-file: 上一步中提示的uuid.json

ingress:
   - hostname: "你的回源域名 比如aaa.com(回退源)"
     service: 你的本地服务地址。比如127.0.0.1:443
     originRequest:
        noTLSVerify: true  
   - hostname: "你的使用SaaS的域名比如 bbb.com 123.bbb.com（自定义主机名）"
     service: 127.0.0.1:443  #同上
     originRequest:
        noTLSVerify: true  #必须为True。因为Cloudflare永远只会验证aaa.com，不开这个选项的话会验证失败。
   - hostname: "使用SaaS的域名也支持通配符。比如 *.ccc.com"
     service: 127.0.0.1:443  #同上
     originRequest:
        noTLSVerify: true

    #这里是Catch All规则，当以上域名都没有匹配上时会执行这段。官方默认是返回404状态。这里我们让它继续处理请求。
   - service: 127.0.0.1:443  #同上
     originRequest:
        noTLSVerify: true
````
## 绑定域名
```` bash
cloudflared tunnel route dns <UUID> <回退源域名>
````
## 运行隧道
```` bash
cloudflared tunnel run <UUID>
````
或者使用systemd运行

## Saas配置

### 首先添加回退源为你**第五步**中tunnel绑定的域名
<img src="/img/saas.png" width="70%" height="70%">  

### 添加自定义主机名为**第四步**中tunnel配置文件第二个绑定的域名
<img src="/img/custom-domain.png" width="70%" height="70%">  

### 随后跟随指引添加验证txt解析  
<img src="/img/saas2.png" width="70%" height="70%">  

最后在你的自定义主机名使用的域名添加指向回退源的CNAME  

## 然后你就可以享受你的Saas了