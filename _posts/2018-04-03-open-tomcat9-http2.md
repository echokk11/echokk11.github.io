---
author: Wuk
layout: post
title: Tomcat9开启HTTP/2.0支持
subtitle: 下一代HTTP协议，大幅度提升web服务性能
catalog: true
tags:
  - HTTP/2
  - tomcat9
  - ssl
---

## HTTP/2
`HTTP/2`基本概念我就不详细阐述了，基本概括为
> 多路复用 (Multiplexing)    
> 多路复用允许同时通过单一的 HTTP/2 连接发起多重的请求-响应消息。

- 这里有一个**Akamai**公司建立的官方演示，[HTTP/2: the Future of the Internet](https://http2.akamai.com/demo)展示出HTTP/2的性能优势
- 这是知乎里关于解释HTTP/2原理的文章，[HTTP/2.0 相比1.0有哪些重大改进？](https://www.zhihu.com/question/34074946)
- `Tomcat`从版本9开始已经支持HTTP/2，同时还有`Jetty`和SpringBoot推荐的容器`Undertow`，我也是从学习`SpringBoot2`过程中了解到的。HTTP/2开启必须SSL，出于安全性考虑吧。

## 生成证书
下载[apache-tomcat9](https://tomcat.apache.org/download-90.cgi)    
解压，假设tomcat根目录表示为$CATALINA_HOME
```bash
cd $CATALINA_HOME/
mkdir ssl
cd ssl
openssl genrsa -out server.key 2048
openssl rsa -in server.key -out server.key
openssl req -new -x509 -key server.key -out ca.crt -days 3650
```

## 修改server.xml
找到如下，打开注释，值得注意的是需要开启APR的支持
```xml
<Connector port="8443" protocol="org.apache.coyote.http11.Http11AprProtocol"
            maxThreads="150" SSLEnabled="true" >
    <UpgradeProtocol className="org.apache.coyote.http2.Http2Protocol" />
    <SSLHostConfig>
        <Certificate certificateKeyFile="ssl/server.key"
                        certificateFile="ssl/ca.crt"
                        type="RSA" />
    </SSLHostConfig>
</Connector>
```

## 安装apr，mac为例，windows只要下载对应的dll文件即可
下载[apr](https://apr.apache.org/download.cgi)
```bash
cd apr
CFLAGS='-arch x86_64' ./configure
make
sudo make install
```
默认安装路径在`/usr/local/apr`

如果没有安装openssl执行这一步
```bash
brew install openssl
```

编译`tomcat-native`
```bash
cd $CATALINA_HOME/bin
tar zxvf tomcat-native.tar.gz
cd tomcat-native-1.2.16-src/native
CFLAGS='-arch x86_64' ./configure --with-apr=/usr/local/apr --with-ssl=/usr/local/opt/openssl --with-java-home=/Library/Java/JavaVirtualMachines/jdk1.8.0_152.jdk/Contents/Home
make
sudo make install
```

最后加链接，不然无效
```bash
sudo ln -s /usr/local/apr/lib/libtcnative-1.dylib /Library/Java/Extensions/
```

启动tomcat
```bash
cd $CATALINA_HOME/bin
./startup.sh
```

查看$CATALINA_HOME/logs目录下以`localhost_access_log.`开头的并且以当天日期结尾的txt日志文件内容，
```txt
::1 - - [03/Apr/2018:18:08:08 +0800] "GET / HTTP/2.0" 200 11228
::1 - - [03/Apr/2018:18:08:08 +0800] "GET /tomcat.css HTTP/2.0" 200 5581
::1 - - [03/Apr/2018:18:08:08 +0800] "GET /tomcat.png HTTP/2.0" 200 5103
::1 - - [03/Apr/2018:18:08:08 +0800] "GET /bg-upper.png HTTP/2.0" 200 3103
::1 - - [03/Apr/2018:18:08:08 +0800] "GET /bg-button.png HTTP/2.0" 200 713
::1 - - [03/Apr/2018:18:08:08 +0800] "GET /bg-middle.png HTTP/2.0" 200 1918
::1 - - [03/Apr/2018:18:08:08 +0800] "GET /asf-logo-wide.svg HTTP/2.0" 200 27235
::1 - - [03/Apr/2018:18:08:08 +0800] "GET /bg-nav.png HTTP/2.0" 200 1401
::1 - - [03/Apr/2018:18:08:09 +0800] "GET /favicon.ico HTTP/2.0" 200 21630
::1 - - [03/Apr/2018:18:08:19 +0800] "GET /tomcat.css HTTP/2.0" 200 5581
::1 - - [03/Apr/2018:18:08:24 +0800] "GET / HTTP/2.0" 200 11228
::1 - - [03/Apr/2018:18:08:24 +0800] "GET /tomcat.png HTTP/2.0" 200 5103
::1 - - [03/Apr/2018:18:08:24 +0800] "GET /tomcat.css HTTP/2.0" 200 5581
::1 - - [03/Apr/2018:18:08:24 +0800] "GET /bg-middle.png HTTP/2.0" 200 1918
::1 - - [03/Apr/2018:18:08:24 +0800] "GET /bg-button.png HTTP/2.0" 200 713
::1 - - [03/Apr/2018:18:08:24 +0800] "GET /bg-upper.png HTTP/2.0" 200 3103
::1 - - [03/Apr/2018:18:08:24 +0800] "GET /asf-logo-wide.svg HTTP/2.0" 200 27235
::1 - - [03/Apr/2018:18:08:24 +0800] "GET /bg-nav.png HTTP/2.0" 200 1401
::1 - - [03/Apr/2018:18:08:53 +0800] "GET / HTTP/2.0" 200 11228
::1 - - [03/Apr/2018:18:08:53 +0800] "GET /tomcat.css HTTP/2.0" 200 5581
::1 - - [03/Apr/2018:18:08:53 +0800] "GET /favicon.ico HTTP/2.0" 200 21630
```
看到HTTP/2.0字样，开启成功
