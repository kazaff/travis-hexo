title:  centos7下安装puppeteer
date: 2021-03-22 09:37:00
tags:
- centos
- puppeteer

categories: 运维
---

刚做了个服务迁移，结果发现比预期想的事儿要多啊，不开心。。
迁移后发现程序报错，提示缺少一大堆库文件。。。记忆里之前也解决过这个问题，搜了一下博客，竟然发现没有记录下来，那好吧，这次就补一下该内容。

其实一般碰到这种缺少库文件的问题时，没什么悬念，安装对应版本即可。
所以我淡定的按照提示缺少的库名，进行了一轮安装：

```
yum install -y at-spi2-atk libXcursor libXdamage cups-libs libXScrnSaver libXrandr atk pango gtk3
```

由于我是在服务器环境下运行的，所以肯定开启的headless模式，所以官方提示需要预装的一些界面库我就不需要了。
有需要的同鞋可以参考下面这个列表：

```
alsa-lib.x86_64
atk.x86_64
cups-libs.x86_64
gtk3.x86_64
ipa-gothic-fonts
libXcomposite.x86_64
libXcursor.x86_64
libXdamage.x86_64
libXext.x86_64
libXi.x86_64
libXrandr.x86_64
libXScrnSaver.x86_64
libXtst.x86_64
pango.x86_64
xorg-x11-fonts-100dpi
xorg-x11-fonts-75dpi
xorg-x11-fonts-cyrillic
xorg-x11-fonts-misc
xorg-x11-fonts-Type1
xorg-x11-utils
```

打完收工~~

### 参考资料

[Chrome headless doesn't launch on UNIX](https://github.com/puppeteer/puppeteer/blob/main/docs/troubleshooting.md#chrome-headless-doesnt-launch-on-unix)
