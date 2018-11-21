---
author: Wuk
date: 2018-04-09 21:22:00+00:00
layout: post
title: 在vultr上搭建科学上网工具shadowsocks
subtitle: 畅游互联网世界
catalog: true
tags:
- vps
- shadowsocks
---

> 上个网都禁锢，什么鬼？

## 硬需求
尤其是编程人员，需要经常用到google搜索信息，也会经常用到国外一些技术网站。

## 方式
翻墙方法也很多，以前的goagent，购买vpn等，考虑到稳定性和个性化需求，完全可以自己在国外买一个VPS配合开源的shadowsocks，在任何端都能实现轻松翻墙。

## vultr
国外VPS选择很多，如linode等。[vultr](https://www.vultr.com/?ref=6891563)是后来建立的公司，性价比很不错，最低配置的主机才2.5刀一个月，但是经常缺货，推荐买5美元的。甚至可以在上面搭建jdk，mysql，tomcat等其他东西也足够自己玩耍。我除了搭建ss翻墙外，还自己搭建了一个[wordpress](https://wordpress.org/)用nginx转发，当作个人博客玩耍，还搭建了[frp](https://github.com/fatedier/frp/blob/master/README_zh.md)做映射工具。

## shadowsocks
貌似debian占用内存少，所以deploy一个debian的vps，节点选择Tokyo，我测过LA等，还是Tokyo稳定
### 安装git
```bash
apt-get update
apt-get install git
```
### Install 
```bash
apt-get install python-pip
pip install git+https://github.com/shadowsocks/shadowsocks.git@master
```
### Usage
```bash
ssserver -p 8388 -k password -m aes-256-cfb
```
To run in the background:
```bash
sudo ssserver -p 8388 -k password -m aes-256-cfb --user nobody -d start
```
To stop:
```bash
sudo ssserver -d stop
```
To check the log:
```bash
sudo less /var/log/shadowsocks.log
```
### Usage with Config File
```bash
vim /etc/shadowsocks.json
```
```javascript
{
    "server":"my_server_ip",
    "server_port":8388,
    "local_address": "127.0.0.1",
    "local_port":1080,
    "password":"mypassword",
    "timeout":300,
    "method":"aes-256-cfb",
    "fast_open": false
}
```
To start:
```bash
ssserver -c /etc/shadowsocks.json
```
此时服务器就会开启一个8388端口，提供ss服务

## 高级
上述搭建成功后，已经基本能满足日常需求，如果还有强迫症精益求精，那么还有方法加速，但这个加速我本人觉得影响不是很大。    
如果是选择debian9操作系统，可以查看内核`uname -a`那么如果已经大于`4.9`，那么后续操作就简单多了

开机后 `uname -r` 看看是不是内核 >= 4.9  

执行 `lsmod | grep bbr`，如果结果中没有 `tcp_bbr` 的话就先执行
```
modprobe tcp_bbr
echo "tcp_bbr" >> /etc/modules-load.d/modules.conf
```

执行
```
echo "net.core.default_qdisc=fq" >> /etc/sysctl.conf
echo "net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.conf
```

保存生效  
`sysctl -p`  

执行  
```
sysctl net.ipv4.tcp_available_congestion_control
sysctl net.ipv4.tcp_congestion_control
```
如果结果都有 `bbr`, 则证明你的内核已开启 bbr  

执行 `lsmod | grep bbr`, 看到有 tcp_bbr 模块即说明 bbr 已启动  

更详细信息，传送门:[开启BBR加速](https://github.com/iMeiji/shadowsocks_install/wiki/%E5%BC%80%E5%90%AFTCP-BBR%E6%8B%A5%E5%A1%9E%E6%8E%A7%E5%88%B6%E7%AE%97%E6%B3%95)

## Client
### 各个平台
- [windows](https://github.com/shadowsocks/shadowsocks-windows)
- [mac](https://github.com/shadowsocks/ShadowsocksX-NG)
- [Android](https://github.com/shadowsocks/shadowsocks-android)，中文名称叫影梭
- ios,以前大陆市场可以购买`surge`,`shadowrocket`，现在只能去美区买，`shadowrocket`性价比更高
### 配置
配置过一个后可以用二维码分享，大致和服务器端差不多就是地址，端口，密码以及加密方法
- PAC方式会有一定的规则，大致是访问不了的走ss
- 全局是全部走ss
- 手机端配置这边有一个rules库不错[Shadowrocket-ADBlock-Rules](https://github.com/h2y/Shadowrocket-ADBlock-Rules)


> 开始浪吧，并不是一劳永逸，也有vps被墙(IP被封)的风险，但比起购买的服务性价比不说，稳定多了吧？