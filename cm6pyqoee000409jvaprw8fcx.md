---
title: "群晖部署alist,cd2,alist2emby,emby及播放302直链视频"
datePublished: Tue Feb 04 2025 04:11:50 GMT+0000 (Coordinated Universal Time)
cuid: cm6pyqoee000409jvaprw8fcx
slug: alistcd2alist2embyemby302
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1738642234161/d43fb849-e704-4dc0-8451-5b01a2b44830.jpeg
tags: 教程

---

一、302是什么

GPT如是说：HTTP 302是HTTP协议中的一种状态码（Status Code），全称是HTTP 302 Found，表示请求的资源临时移动到了一个新的URL，即临时重定向。当浏览器或其他客户端向服务器发送请求时，如果服务器返回一个302状态码，它会同时在响应头中提供一个Location字段，告知客户端资源现在可以在新的URL找到。客户端随后应该使用这个新的URL来重新发起请求。这种重定向通常是临时的，意味着资源只是暂时可以在新的地址找到，未来还可能会返回到原来的地址。

在此教程中，利用Alist的获取115网盘直链的功能，也就是302跳转直链，来实现Emby播放时直连网盘地址，不走Emby所在网络上下行，减少了一次中转，达到外网高带宽看片的需求。

二、目前已测试以下客户端播放可以实现302跳转

Emby客户端  Infuse Fileball  Vidhub  Yamby

欢迎补充…

三、准备条件：

1、有空间及会员的115账号，CD2(CloudDriver2)，可以不需要会员，Alist

，Emby，可以选择自己喜欢的镜像版本，如emby/embyserver、linuxserver/emby、amilys/emby、nginx

2、ssh命令安装容器前，需要先建立对应的文件夹，例如cd2标注黄色的的这两个文件夹

\-v /volume1/docker/clouddrive/Config:/Config \\

\-v /volume3/video:/downloads:shared \\ssh命令安装的过程中，有可能会提示错误，遇到的有端口冲突的，和格式错误，可以复制错误给gpt，根据提示改正即可。

3、alist域名访问的时候需要把端口反代一下

4、ssh的安装容器命令和软件，群辉有备份

5、参考的教程[https://blog.nanako.vip/?p=162](https://blog.nanako.vip/?p=162)，以实际配置过程为准

四、配置过程

1、安装alist容器

docker run -d --restart=always \\

  -v /volume1/docker/alist:/opt/alist/data \\

  -p 5244:5244 \\

  -e PUID=0 \\

  -e PGID=0 \\

  -e UMASK=022 \\

  --name="alist" \\

  xhofe/alist:latest

启动以后，点击添加，选择115网盘

![](https://img1.warm.us.kg/img1/main/images/202502032321166.png align="left")

填写如图的这些信息，难点是填cookie，可以通过常规方式登录115网盘网页抓取cookie，也可以[https://alist.nn.ci/zh/guide/drivers/115.html](https://alist.nn.ci/zh/guide/drivers/115.html)这个教程里面的方法，QRCode 扫码方式登录，（抓取的api填到二维码令牌，保存自动就有cookie了）。闲鱼卖家教了另外一种抓115小程序的cookie的方法（尽量不要用115小程序，就不容易掉cookie），方法：安装reqable软件（备份了），企业微信登录115小程序，开启reqable，抓取cookie。

![](https://img1.warm.us.kg/img1/main/images/202502040919684.png align="left")

播放时候卡，改成o，不知道什么原理，记得改一下就行

![](https://img1.warm.us.kg/img1/main/images/202502040920052.png align="left")

2、安装cd2容器

利用ssh命令安装，安装命令如下

docker run -d \\

    --name clouddrive \\

    --restart unless-stopped \\

    --env CLOUDDRIVE\_HOME=/Config \\

    --env TZ=Asia/Shanghai \\

    --env PUID=0 \\

    --env PGID=0 \\

    -v /volume1/docker/clouddrive/Config:/Config \\

    -v /volume3/video:/downloads:shared \\

    --network host \\

    --pid host \\

    --privileged \\

    --device /dev/fuse:/dev/fuse \\

    cloudnas/clouddrive2

安装好以后配置参数，配置如图webdav

![](https://img1.warm.us.kg/img1/main/images/202502040921855.png align="left")

点击以后，输入alist的域名地址[http://loveximi.cy0113.top:3255/dav](http://loveximi.cy0113.top:3255/dav)（别忘了有/dav，然后原本端口是3244，需要反代一下3255或其他 ），账号admin和密码alist容器安装的时候在日志里面可以看到，如图

![](https://img1.warm.us.kg/img1/main/images/202502040922687.png align="left")

把账号地址输入正确以后就成功了一大半了，如图挂载这一个

![](https://img1.warm.us.kg/img1/main/images/202502040923107.png align="left")

3、安装emby

利用ssh命令安装，安装命令如下

docker run \\

  --network=bridge \\

  -p '8097:8096' \\

  -p '8920:8920' \\

  -p '1907:1900/udp' \\

  -p '7358:7358/udp' \\

  -v /volume1/docker/love-emby:/config \\

  -v /volume3/video:/downloads:rslave \\

  -e TZ="Asia/Shanghai" \\

  --device /dev/dri:/dev/dri \\

  -e UID=1024 \\

  -e GID=100 \\

  -e GIDLIST=0 \\

  -e ALL\_PROXY=http://192.168.8.200:7890 \\

  -e NO\_PROXY=172.17.0.1,127.0.0.1,localhost \\

  --restart always \\

  --name emby \\

  -d lovechen/embyserver:latest

这个没啥好说的，主要注意安装过程中，端口冲突就改一下。

4、Nginx

操作到这里，你需要自己明确一下Emby和Alist的对应文件路径。

两边对比一下同一个视频文件，可以看到，Emby中的路径应为/downloads/115/影视/emby/电影/动画电影/灌篮高手 (2022)，Alist中的同一个文件路径为/115/影视/emby/电影/动画电影/灌篮高手 (2022)

具体的网盘内路径以自己为准，但如果完全照抄以上配置的话，路径的开头/downloads/115 及 /115 肯定是和我一致的。

安装nginx前，下载一个nginx的脚本（以后用不了可能还需要重新找），修改需要自定义配置的有两个文件。

conf.d/constant.js

画红线几个地方是要修改的，embyHost: Emby的内网地址及端口

embyMountPath: 填写上述的 /downloads ，即Alist路径为Emby的子集剩余的部分。注意填写到此列表变量的元素中，即 \["/downloads"\]。如果你的Emby文件路径与Alist完全一致，即剩余部分为空，那么此处填写 \[\]。

embyApiKey: Emby的ApiKey，从Emby的web中获取。

alistAddr: Alist的地址及端口。填写内网地址。

AlistpublicAddr的地址及端口。填写外网的地址。

alistToken: Alist的Token令牌，在 Alist -&gt; 管理 -&gt; 设置 -&gt; 其他 -&gt; 令牌 获取，格式为 alist- 开头

![](https://img1.warm.us.kg/img1/main/images/202502040925769.png align="left")

conf.d/emby.conf

第26行: listen 8091;

此为nginx监听的端口，修改为你想要的端口即可，用于替代原Emby的8097端口访问。访问此端口才能实现302转发；修改完配置之后，将conf.d 及 nginx.conf 映射到nginx容器，在上述的nginx的ssh命令配置中已添加，注意存放的宿主机路径。启动nginx容器。**<mark>图片标红的可改可不改，我用的默认的</mark>**<mark>。</mark>

![](https://img1.warm.us.kg/img1/main/images/202502040927732.png align="left")

利用ssh命令安装，安装命令如下

docker run --name nginx \\

\&gt;   --restart unless-stopped \\

\&gt;   -v /volume1/docker/nginx/conf.d:/etc/nginx/conf.d \\

\&gt;   -v /volume1/docker/nginx/nginx.conf:/etc/nginx/nginx.conf \\

\&gt;   -v /volume1/docker/nginx/emby:/var/cache/nginx/emby \\

\&gt;   --network host \\

\&gt;   nginx:latest

5、测试

此时启动Nginx容器后，访问 8091 端口，或是你在 emby.conf 文件中修改的端口，可以访问到Emby Web。即完成了第一步，nginx代理Emby Web成功。

方法一：使用各类客户端，如Emby官方客户端、Infuse、Fileball、Vidhub等登录8091端口，找一个刮削好的片子播放，或者web跳转第三方播放器potplayer、iina等，查看CD2的下载任务。如果没有大流量的对应文件下载，进程为 /system/EmbyServer ，即302转发成功。

方法二：浏览器右键-选择检查 -再选择网络  随便打开一个片子就可以了

网络就是那个长得像WiFi的按钮，链接是这样就代表成功302直链了https://cdnfhnfile.115.com/65e96c5d4fda30ac7069812fe9a67f7125d4dc7c/%E7%81%8C%E7%AF%AE%E9%AB%98%E6%89%8B.The%20First%20Slam%20Dunk.2022.2160p.BluRay%20DV%20UHD.TrueHD%207.1%20Atmos-HDS.mkv?t=1716076837&u=360200100&s=524288000&d=vip-1864197998-cwgqo1lcugg81dzgz-1&c=2&f=1&k=89b39a1733b967b922e2e7d01b438f1b&us=5242880000&uc=10&v=1