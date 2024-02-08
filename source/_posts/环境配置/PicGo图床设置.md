---
title: PicGo图床设置
date: 2024-02-08 17:52:47
tags:
  - PicGo
  - 图床
categories:
  - 环境配置
---

## 图床的介绍

图床在台湾叫做 `網路相簿`，实际上就是可以将图片存放在网络上。好处在于比如在 markdown 中插入图片，假如使用本地图片的方式，那么当博客的位置发生变更时图片将会失效。尽管可以通过相对路径的方式部分程度的进行解决，但是假如想要将 markdown 文件分享给别人，如果想要他获得相同的显示效果就必须把图片连同 markdown 文件一起分享出去。而使用图床就可以做到 markdown 的显示效果在各个地方都是相同的，并且减少空间占用。

<!-- more -->

## PicGo 图床

### 安装

目前在国内 `PicGo` 图床是一个比较好用的跨平台图床软件，并且支持多种图床。我使用的是 `windows 2.3.1` 版本的 PicGo，打开[下载地址](https://github.com/Molunerfinn/PicGo/releases/tag/v2.3.1) 下载安装包。
![image.png|900]( https://raw.githubusercontent.com/slam-learner/Figures/main/PicGo/202402081807311.png )

按照提示进行安装即可，可以修改安装位置到自己想要的目录下。

### GitHub 存放图片

由于篇幅的原因，可以参考 [2022【保姆级教程】手把手带你使用 PicGo+Github 搭建自己的免费图床](https://zhuanlan.zhihu.com/p/553533337) 进行操作，主要的步骤就是新建仓库-生成 token-填入 PicGo 的设置中。并且可以设置 PicGo 的一些选项，我的设置如下：
![image.png|500](https://raw.githubusercontent.com/slam-learner/Figures/main/PicGo/202402081815752.png)

### Obsidian 设置

我使用的本地笔记软件是 Obsidian，可以通过安装`image auto upload`插件实现粘贴图片时默认上传到 PicGo 图床中，非常的方便。

## 参考资料

1. [知乎：2022【保姆级教程】手把手带你使用 PicGo+Github 搭建自己的免费图床](https://zhuanlan.zhihu.com/p/553533337)
2. [知乎：picGo+gitee 搭建 Obsidian 图床，实现高效写作](https://zhuanlan.zhihu.com/p/427353062)
