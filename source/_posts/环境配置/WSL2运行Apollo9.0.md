---
title: WSL2运行Apollo9.0
date: 2024-04-12 16:59:11
tags:
  - Apollo
  - WSL2
categories:
  - 环境配置
---

## 参考资料

1. [Apollo官方文档-源码安装说明](https://apollo.baidu.com/docs/apollo/9.0/md_docs_2_xE5_xAE_x89_xE8_xA3_x85_xE6_x8C_x87_xE5_x8D_x97_2_xE6_xBA_x90_xE7_xA0_x81_xE5_xAE_x89_c6622abe953930b2197ae66df0a51dcd.html)
2. [Apollo官方社区文章-40系列显卡新镜像支持](https://apollo.baidu.com/community/article/1212)
3. [Apollo Github Issue-14821 使用4090显卡编译报错](https://github.com/ApolloAuto/apollo/issues/14821)
4. [Apollo Github Issue-14478 Fail to build apollo in WSL2 ( error code: 14, error message: 'Socket closed')](https://github.com/ApolloAuto/apollo/issues/14478)
5. [Apollo Github Issue-289 WSL2: nvidia-container-cli mount error, libnvidia-ml.so.1: file exists: unknown](https://github.com/NVIDIA/nvidia-container-toolkit/issues/289)
6. [Apollo Github Issue-15246 dev_start进不去
](https://github.com/ApolloAuto/apollo/issues/15246)

## 安装步骤

首先根据官方文档安装好依赖，然后下载`Apollo v9.0.0`源码，解压后进入`apollo`目录。

### 1. 修改`dev_start.sh`脚本

Apollo默认镜像对于40系列显卡不支持，根据参考资料2，需要修改`docker/scripts/dev_start.sh`中的`VERSION_X86_64`镜像版本:

```text
VERSION_X86_64="dev-x86_64-18.04-20231128_2222"
```

按照参考资料2中的步骤说明，下一步直接执行`./docker/scripts/dev_start.sh`进行镜像构建即可，但在`WSL2`下构建会出现以下问题：

```text
nvidia-container-cli mount error, libnvidia-ml.so.1: file exists: unknown.
```

实际上镜像已经成功生成，只是在执行容器时出现错误，根据参考资料5和参考资料6，首先将`dev_start.sh`中的容器执行命令注释掉。

然后按照以下步骤进行操作：

```bash
# 查看docker 镜像id
docker images
# 进入容器
sudo docker run -it --rm <Apollo的镜像ID>
# 删除镜像内NVIDIA相关文件
rm /usr/lib/x86_64-linux-gnu/libnvidia-*
rm /usr/lib/x86_64-linux-gnu/libcuda.so*
rm /usr/lib/x86_64-linux-gnu/libnvcuvid.so.*
