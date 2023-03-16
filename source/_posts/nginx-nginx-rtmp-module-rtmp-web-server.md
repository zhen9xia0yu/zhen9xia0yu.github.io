---
title: 使用nginx+nginx-rtmp-module搭建rtmp流服务器
categories:
- code
tags:
- nginx
- rtmp
---

最近要做一个rtmp流转rtp的项目，需要一个固定可用的rtmp流源。公网上流传的可用rtmp流地址太不稳定，央视的最近好几个月都没连上过，不知道是不是一些敏感的zz问题又关闭一个access...当然了，可能纯粹是换了src :)，而用外网的直播源，也很难验证音画同步的问题。

嗖，自己用nginx搭一个吧，还能控制codec，尽在掌握了。👏

## 准备工作

1. nginx-version.tar.gz
   * 可以直接在官网下载最新版本 <https://nginx.org/en/download.html>
2. nginx-rtmp-module    
   * 库地址 <https://github.com/arut/nginx-rtmp-module>

## 编译安装

.
|-- nginx-version.tar.gz
|-- nginx-rtmp-module

```bash
tar -zxvf nginx-verison.tar.gz
cd nginx-version
./configure --prefix=$NGX_PATH --add-module=../nginx-rtmp-module
make && make install
```
## 配置启动

以下操作中可能会遇到的权限、端口防火墙问题等就不再赘述了。

```bash
vim $NGX_PATH/conf/nginx.conf
```

1. 添加与http模块同级的rtmp模块
   * application的名字没有讲究，区别在于其的用法，play是用来直接播放指定媒体文件的；live则是push/pull rtmp stream service
   * 添加完这个模块后，流媒体服务就可以开启了，但若想要能实时查看到两个service的流媒体数据，则继续进行第二步。

```nginx
rtmp {
    server {
        listen 1935; #rtmp流服务监听端口
        chunk_size 4096; #分chunk发送数据size

        application media4tst { #play service
            play /home/zhengxiaoyu/media4tst; #$MEDIA_PATH，后文会讲
        }

        application live { #live service
            live on;
        }
    }
}
```

2. 在http模块的server部分中添加内容

```nginx
    server {
        listen       10300; #nginx的http请求监听端口，默认是80
        server_name  localhost;
    
        location /stat{ #当前rtmp流数据情况
            rtmp_stat all;
            rtmp_stat_stylesheet stat.xsl;
        }
        location /stat.xsl { #显示格式
            root /home/zhengxiaoyu/nginx_source/nginx-rtmp-module/;
        }
    #已省略其他无关内容
}
```

3. 启动、更新、停止

```bash
./$NGX_PATH/sbin/nginx #启动
./$NGX_PATH/sbin/nginx -s reload #重启，使用的是默认配置文件
./$NGX_PATH/sbin/nginx -c xxx.conf #指定配置文件启动
./$NGX_PATH/sbin/nginx -s stop #停止
```

## 测试运行

1. 检验nginx服务是否启动

   ![image]( sendpix1.jpg) 

2. 检验rtmp stat是否可用

![image]( sendpix3.jpg) 

3. 检验rtmp-play是否可用

   * 在$MEDIA_PATH下存放媒体文件

   ```bash
   /home/zhengxiaoyu/media4tst/iphone11cast.mp4
   ```

   * 使用ffplay播放目标文件

   ```bash
   ffplay rtmp://127.0.0.1/media4tst/iphone11cast.mp4 #127.0.0.1:1935 also useful.
   ```

   此时刷新<localhost:10300/stat>，也可看到流的具体信息。

   ![image]( 1635853587187-1635856625630.png) 

4. 检验rtmp-live是否可用

   * ffmpeg push rtmp stream

   ```bash
   ffmpeg -re -stream_loop -1 -i iphone11cast.mp4 -vcodec copy -acodec copy -f flv rtmp://127.0.0.1:1935/live/zxy
   ```

   ![image]( sendpix4-1635856708398.jpg) 

   * ffplay pull rtmp stream

   ```bash
   ffplay rtmp://127.0.0.1:1935/live/zxy
   ```

   ![image]( 1635853837040.png) 

-------

嗯，差不多就是这样了。
