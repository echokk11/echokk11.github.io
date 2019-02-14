---
author: Wuk
layout: post
title: 利用gitlab的webhooks自动触发jenkins-ci构建
subtitle: 也顺便说说gitlab-ci
catalog: true
tags:
  - gitlab-ci
  - webhooks
  - jenkins
---

> 不知道怎么用CI工具，还敢自称code dog？


## 缘由
我也是几年前才接触CI/CD的概念，中文名称叫持续集成，具体是什么就自行了解吧。总之就是能自动化完成都自动化完成，你写好代码，提交代码，什么编译拉，打包，上传，重启等，都是可以一键自动完成，更厉害的连这个"一键"都可以省去。要充分运用好这个工具，有一点linux的`shell`知识，就更加妙不可言了。

## gitlab-ci
其实简单的构建我本人是更喜欢`gitlab-ci`的，只要在目标机器上安装并注册[gitlab-runner](https://docs.gitlab.com/runner/)，然后只要在项目根目录下建立一个叫`.gitlab-ci.yml`的标记语言，写下简单的如下内容
```yaml
stages:
  - install_deps
  - test
  - build
  - deploy_test
  - deploy_production
    
build:
  stage: build
  only:
    - dev
    - master
  script:
    - 'whoami'
    - 'pwd && mvn clean deploy -Dmaven.test.skip=true'
```
每一次提交(push)gitlab都会根据检查这个文件来验证是不是满足触发条件，然后就会执行`script`下的内容，每一行`-`后面的内容相当于写shell。    
stages看自己定义，基本上可以分为这些，如果只做一件事情，如上面的例子，其实都可以省去留一个`build`    
`only`一眼就看出只有dev和master分支才符合触发条件，我这边触发后只是简单的利用`maven`deploy相关jar包到对应仓库    
其实`gitlab-ci`能做的远远不止如此。    
> 就算`github`也有[travis-ci](https://www.travis-ci.org)大法用。这几天是`github`成立整整十周年之际，他见证了无数优秀技术，代码框架的崛起，一把眼泪一把涕。    
耳熟能详的有    
- 2009年发布的,并在2010年在github上开源的`Bitcoin`
- 2009年5月,`Node.js`发布
- 2014年6月,`Docker`1.0发布,创建仓库是在2013年
- 2014年7月,微软的`.NET`开源
- 2015年12月,苹果开源`Swift`
- 2016年7月,`Apollo-11`开源，了解一下60年代的程序员的编码风格，如何送你上月球的
- 2017年2月,`TensorFlow`1.0发布，AI越来越近    
以上只是冰山一角，可见`github`的影响力，跑题了。

## webhooks
`gitlab`还提供了webhooks功能，意思就是仓库中有例如push，merge，issue等事件发生后，会往指定的web地址发送数据包，而构建神器jenkins安装插件后整好支持这个功能。
- 首先jenkins要安装两个插件`gitlab plugin`和`gitlab hook plugin`。系统管理->管理插件，选择可选插件，然后在右上角搜索两个插件安装。
- 这个时候找到相关的project，配置，找到`构建触发器`，勾起`Build when a change is pushed to GitLab. GitLab webhook URL:`，并且复制下后面的url地址    
![img](/img/in-post/gitlab-hooks-with-jenkins/QQ20180416-190214@2x.png)
- 点击高级继续设置，需要点击`Generate`生成`token`，其余几个选项，默认是全部分支构建，你也可以用分支名称，甚至正则表达式指定分支构建。    
![img](/img/in-post/gitlab-hooks-with-jenkins/QQ20180416-192326@2x.png)    
还有更有意思的的是你看到上面的`Comment (regex) for triggering a build`没，默认值是`Jenkins please retry a build`，意思就是如果没有设置push event触发构建，那么把webhook里的`Comments`勾起来，只要内容是`Jenkins please retry a build`，一样会触发构建。
- 随后去gitlab对应的项目下，找到`Settings`->`Integrations`，点击Add webhook,填写好上面的URL和token        
![img](/img/in-post/gitlab-hooks-with-jenkins/QQ20180416-191450@2x.png)
- 点击test->Push events测试，上方出现蓝色背景的字样`Hook executed successfully: HTTP 200`就说明已经配置成功了。    
![img](/img/in-post/gitlab-hooks-with-jenkins/QQ20180416-191751@2x.png)
- 最后去jenkins查看构建历史，发现已经有了    
![img](/img/in-post/gitlab-hooks-with-jenkins/QQ20180416-193249@2x.png)

## systemhooks
gitlab还有一个`System Hooks`，(需要管理员权限)，比如在创建，修改仓库等一些系统级事件改变时候触发用的，对于编码者来言，用的频率并不高。
