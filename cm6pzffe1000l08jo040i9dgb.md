---
title: "(全)轻松流播：掌握 Emby、CD2、Alist、STRM 和 302 重定向，实现无缝观看"
seoTitle: "无缝观看指南：Emby 与流媒体工具"
seoDescription: "了解如何使用 Emby、CD2 和 STRM 实现轻松流媒体，通过 302 重定向快速访问云盘内容。"
datePublished: Tue Feb 04 2025 04:31:04 GMT+0000 (Coordinated Universal Time)
cuid: cm6pzffe1000l08jo040i9dgb
slug: embycd2aliststrm-302
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1738642699817/4f241c4e-a2bf-4d8f-89b7-e65b2bc3a3f4.jpeg
tags: 302, 教程, emby

---

## **前言**

什么是 STRM 文件 ？STRM 文件就像是网盘文件的快捷方式，生成后保存在本地。 它是一种简单的文本文件，指向网盘或本地文件的位置，而不存储文件本身。通过 STRM 文件，媒体服务器（如 Emby）可以直接读取链接并播放网盘上的内容，无需下载整个文件到本地。

auto\_symlink 3.1 作用 实时监控目录变化，扫盘，入库，能够将123网盘的视频生成以本地 .strm 文件的形式保存，还能确保这些文件被EMBY正确识别。当通过支持EMBY的客户端添加EMBY反代地址后，配合emby2alist播放视频时会直接从 .strm 文件中读取直连链接，从而连接到123服务器进行播放。 123直连的优势 视频播放时直接连接网盘服务器，无需经过本地设备处理，这不仅避免了占用本地网络的上行带宽，还能在网络带宽充足的情况下实现流畅播放。此外，使用影库设备进行刮削操作时，也不会触发123的风控机制，从而提供更安全可靠的使用体验。温馨提示：目前CMS方案，大多数软件可能不支持ISO播放。群晖上安装好moviepolit-v2，下载好视频上传刮削到123网盘。

1. 想要在不下载的情况下无缝串流您喜爱的电影吗？了解 STRM 文件如何作为云存储的快捷方式，彻底改变您的媒体体验。
    
2. STRM 文件允许像 Emby 这样的媒体服务器直接读取和播放云存储中的内容，无需进行本地下载。这意味着流媒体播放更流畅，带宽占用更少。
    
3. 使用 auto\_symlink 3.1，您可以实时监控目录变化，确保媒体库始终是最新的，并可通过 Emby 访问，而不会触发安全机制。
    
4. 在开始之前，请确保您拥有云服务器、CD2 和 123 网盘账号。将 CD2 与您的云存储集成，实现无缝访问。
    
5. 使用 Docker 在您的服务器上安装 CD2、Alist、Emby 和 auto\_symlink。只需确保端口不冲突，您就可以开始了。
    
6. Emby2Alist 使用 Nginx 重定向资源路径，为云存储资源启用 302 重定向。此设置确保您的媒体始终可访问。
    
7. 通过配置 Nginx 设置来部署 Emby2Alist。设置完成后，通过服务器的 IP 访问并轻松开始流播。
    
8. 在群晖上安装 moviepolit-v2，以管理您的媒体上传，并确保它们在云存储中正确分类。
    
9. 遇到 115 资源无法播放的问题？检查您的 Nginx 日志，并调整 constant-mount.js 中的规则以暂时解决。
    
10. 添加了新的云存储但无法在 Emby 上播放媒体？确保您的 Emby 容器的路径映射设置为 rslave 模式以获得正确访问。
    
11. 如果 Emby 显示“无兼容的流”，请验证您的云存储是否正确挂载并可访问。这通常可以解决问题。
    
12. 想深入了解无缝流播的更多信息？阅读我在 @hashnode 上的完整文章，发现掌握您的媒体设置的所有技巧和窍门。阅读更多！
    

# 一、准备工作 在开始配置之前，请确保以下设备和账户已准备就绪：

1.一台云服务器

2.cd2和123网盘账号会员

3.把cd2的webdave上添加123网盘（alist要把123网盘放在主目录）

# 二、安装步骤

1、进入云服务器，直接通过以下命令安装即可，直接可以安装好cd2，alist，emby，autosumlink这4个容器只需要注意端口不要冲突，冲突了就更好端口。

```plaintext
#!/bin/bash

# 创建所需的目录
echo "Creating required directories..."
mkdir -p /media/docker/alist
mkdir -p /media/docker/clouddrive/Config
mkdir -p /media/docker/movie/{cloud_media,webdav_media,strm,webdav_strm}
mkdir -p /media/docker/auto_symlink/config
mkdir -p /media/docker/movie/cloud_media/clouddrive
mkdir -p /media/docker/movie/webdav_media/clouddrive
mkdir -p /media/docker/movie/emby/config

# 启用 MountFlags 共享，用于支持 Fuse3 挂载
echo "Enabling shared mount flags for Docker..."
sudo mkdir -p /etc/systemd/system/docker.service.d/
sudo cat <<EOF > /etc/systemd/system/docker.service.d/clear_mount_propagation_flags.conf
[Service]
MountFlags=shared
EOF
sudo systemctl restart docker.service

echo "Environment setup completed."
```

### 启动容器

以下是`docker-compose.yml`文件的内容，用于启动所有服务（服务器需要能拉取镜像（不管是使用国外服务器还是配置镜像源的方式），或者可以使用`docker load -i`导入镜像的tar包）：

```plaintext
version: "3"

services:
  alist:
    image: xhofe/alist-aria2:latest
    container_name: alist
    restart: always
    environment:
      - PUID=0
      - PGID=0
      - UMASK=022
      - TZ=Asia/Shanghai
    ports:
      - "5244:5244"
      - "6800:6800"
    volumes:
      - /media/docker/alist:/opt/alist/data

  clouddrive:
    image: cloudnas/clouddrive2
    container_name: clouddrive
    restart: unless-stopped
    environment:
      - CLOUDDRIVE_HOME=/Config
    ports:
      - "8097:19798"
    volumes:
      - /media/docker/movie/cloud_media:/media:shared
      - /media/docker/movie/webdav_media:/webdav_media:shared
      - /media/docker/clouddrive/Config:/Config
    privileged: true
    pid: host
    devices:
      - /dev/fuse:/dev/fuse

  auto_symlink:
    image: shenxianmq/auto_symlink:latest
    container_name: auto_symlink
    restart: unless-stopped
    environment:
      - TZ=Asia/Shanghai
    ports:
      - "8095:8095"
    volumes:
      - /media/docker/movie/cloud_media/clouddrive:/media/docker/movie/cloud_media/clouddrive:rslave
      - /media/docker/movie/webdav_media/clouddrive:/media/docker/movie/webdav_media/clouddrive:rslave
      - /media/docker/movie/strm:/media/docker/movie/strm
      - /media/docker/movie/webdav_strm:/media/docker/movie/webdav_strm
      - /media/docker/auto_symlink/config:/app/config
    user: "0:0"

  emby-local:
    image: amilys/embyserver
    container_name: emby-local
    restart: unless-stopped
    environment:
      - UID=0
      - GID=0
      - GIDLIST=0
      - TZ=Asia/Shanghai
    ports:
      - "8096:8096"
    volumes:
      - /media/docker/movie/emby/config:/config
      - /media/docker/movie/cloud_media/clouddrive:/media/docker/movie/cloud_media/clouddrive:rslave
      - /media/docker/movie/webdav_media/clouddrive:/media/docker/movie/webdav_media/clouddrive:rslave
      - /media/docker/movie/webdav_strm:/webdav_strm
      - /media/docker/movie/strm:/strm
    privileged: true
    devices:
      - /dev/dri:/dev/dri
```

运行以下命令来准备环境并启动所有服务：

```plaintext
# 赋予脚本执行权限
chmod +x setup.sh

# 执行脚本
./setup.sh

# 启动容器
docker-compose up -d
```

**2.emby2alist**

## 2.1 作用

* 通过nginx重定向资源地址，将原先strm文件指向的挂载路径如`/media/docker/movie/cloud_media/clouddrive/123云盘/影视资源/电影/外语电影/泳者之心 (2024) {tmdb-774531}/Media/cloud_media/115` 直接指向 alist路径下的`/12123云盘/影视资源/电影/外语电影/泳者之心 (2024) {tmdb-774531}/Media/cloud_media/115`，从而实现302重定向网盘资源
    

## 2.2 部署

```bash

wget https://github.com/bpking1/embyExternalUrl/archive/refs/tags/v0.4.5.tar.gztar -zxvf v0.4.5.tar.gz
cd embyExternalUrl-v0.4.5/
cd emby2Alist

# modify nginx/conf.d/constant.js
# 参考下图1配置
# embyHost如果是本地docker部署的，就是图中的http://172.17.0.1:8096(端口自行确认)

# modify nginx/conf.d/config/constant-mount.js
# 参考下图2配置
# alistAddr如果是本地docker部署的，就是图中的http://172.17.0.1:5244(端口自行确认)

# modify nginx/conf.d/config/constant-pro.js
# 参考下图3配置
# 待替换路径需确认emby媒体库内资源路径和alist内的路径，填写多出来的部分(仅限支持302重定向的网盘资源)
```

nginx/conf.d/constant.js

![](https://img1.warm.us.kg/img1/main/images/202502041116570.png align="left")

nginx/conf.d/config/constant-mount.js

![](https://img1.warm.us.kg/img1/main/images/202502041117481.png align="left")

nginx/conf.d/config/constant-pro.js

![](https://img1.warm.us.kg/img1/main/images/202502041119307.jpeg align="left")

### ***<mark>参考以后的实际设置如图（主要就是把这个设置好，这个设置好就没啥难度了）</mark>***

![](https://img1.warm.us.kg/img1/main/images/202502041121249.png align="left")

2.2.1进入文件夹

cd /media/docker/embyExternalUrl-0.4.5/emby2Alist/docker/

docker-compose up -d

45.207.208.22:8091访问即可（默认端口是8091）

3.群晖上安装moviepolit-v2

安装步骤省略，只要设置好复制到123网盘路径就可以，提醒要打开按类别分类按钮，保存设置。

![](https://img1.warm.us.kg/img1/main/images/202502041122881.png align="left")

## 4.3 常见问题（constant-pro.js设置好以后，通过123网盘一把就安装好了）

### 4.3.1 115资源无法播放

nginx-emby容器内日志如下：

```bash
[warn] 21#21: *155 js: redirect to: /d/115/xxx
```

#### 4.3.1.1 原因

匹配到emby2alist里的115规则，并使用alist公网地址进行转发

![](https://img1.warm.us.kg/img1/main/images/202502041123645.jpeg align="left")

#### 4.3.1.2 临时解决方案

屏蔽位于`constant-mount.js`内`clientSelfAlistRule`中与115有关的规则

![](https://img1.warm.us.kg/img1/main/images/202502041124006.png align="left")

### 4.3.2 新增网盘后emby无法播放

问题描述：在上面的环境正常运转的情况下(emby启动后)，新增了一个夸克盘通过`alist` → `cd2`挂载到本地目录`/volume2/Media/webdav_media`，可以正常生成新的strm文件，emby也可以正常入库，但是无法播放，并提示无兼容的流。此时在emby中查看挂载在emby容器上的 `/volume2/Media/webdav_media`路径，里面的夸克盘路径下是没有东西的

复现方式：

* 在cd2内卸载之前的夸克盘挂载
    
* 去emby内看路径`/volume2/Media/webdav_media`下是否已经没有文件夹
    
* 在cd2内重新挂载夸克盘
    
* 去emby内确认路径`/volume2/Media/webdav_media`下是否虽然有夸克盘的目录，但里面没东西
    

#### 4.3.2.1 原因

这是因为在之前的步骤中，emby容器配置中与网盘相关的路径映射如`cloud_media`, `webdav_media`没有开启rslave模式，现已修正

#### 4.3.2.2 解决方案

emby容器与网盘相关的路径挂载时开启rslave模式，详见4.1

以下操作可保留原先所有配置和刮削记录

```bash
# add :rslave to docker-compose.yml 按照4.1emby部署的compose文件更改
docker stop emby_container # emby_container替换成自己的容器名
docker rm emby_container # emby_container替换成自己的容器名
docker-compose up -d # 重新部署
```

### 4.3.3 emby提示无兼容的流

#### 4.3.3.1 原因

该问题一般由指定路径下的文件不存在引起，可以检查下网盘是否正确挂载

---

参考教程：  
[邱小晨博客](https://www.nerocats.com/archives/58/)