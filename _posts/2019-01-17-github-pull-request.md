---
author: Wuk
layout: post
title: "在github上提交pull request"
subtitle: "以及如何更新fork的的项目"
catalog: true
header-style: text
tags:
  - github
  - "pull request"
  - fork
---

> 会在github上提交PR，也是开源精神中，奉献，共享的的体现，说不定我们也会成为apache项目的commiter呢？


## fork项目的更新
在github上fork一个项目后，源项目有更新，如何把自己的fork项目也更新到最新是我们经常遇到的场景。

1. 点击fork的项目左上角的`New pull request`
![img](/img/QQ20190117-153711@2x.jpg)

2. 点击`switching the base`，反向对比项目
![img](/img/QQ20190117-154255@2x.jpg)

3. 此时应该可以看到源项目较于我们fork的项目的新增commit了，github会检测是否可以merge，点击`Create pull request`按钮，填写title，如
update，最后点击右下角`Create pull request`
![img](/img/QQ20190117-154631@2x.jpg)
![img](/img/QQ20190117-154854@2x.jpg)

4. 在最后的界面，点击`Merge pull request`后，再次点击`Confirm merge`即可。
![img](/img/QQ20190117-155125@2x.jpg)


## 给开源项目提交pull request
其实原理同上，只是不要反向对比。
1. fork你要提交pull request的项目
2. clone到本地修改对应的内容，如果需要，checkout你要提交代码的分支。
3. commit and push到github
4. 点击fork项目的左上角的`New pull request`，后续步骤和fork的更新相同（除去`switching the base`）

[官方说明文档在此，点击查看！！！](https://help.github.com/articles/creating-a-pull-request/)
