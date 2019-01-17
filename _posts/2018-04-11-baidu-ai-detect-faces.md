---
author: Wuk
layout: post
title: 纯js玩转人脸识别并标识出关键点
subtitle: 利用百度AI识别人脸
tags:
  - 人脸识别
  - canvas
---

> 受到深度学习和神经网络的影响，人脸识别正在普及，到底是怎么个东西？

## 是时候展现真正的技术了。

[下载代码，即刻开始](https://github.com/echokk11/face-detect)

## face-detect

1. [百度AI](https://console.bce.baidu.com/ai/?_=1523436939436&fromai=1#/ai/face/overview/index)
先去这里创建应用，获得授权的`API_KEY`和`SECRET_KEY`

2. 修改`App.vue`中的`API_KEY`和`SECRET_KEY`为你自己申请的

## 开始玩
``` bash
# clone
git clone https://github.com/echokk11/face-detect.git

cd face-detect

# install dependencies
npm install

# serve with hot reload at localhost:8080
npm run dev
```

## 跨域
已经在`config/index.js`中增加proxyTable配置，实现代理跨域
```javascript
proxyTable: {
    '/oauth/**': {
        target: 'https://aip.baidubce.com',
        changeOrigin: true
    },
    '/rest/**': {
        target: 'https://aip.baidubce.com',
        changeOrigin: true
    }
}
```

## canvas描框，描点
看js代码实现吧
