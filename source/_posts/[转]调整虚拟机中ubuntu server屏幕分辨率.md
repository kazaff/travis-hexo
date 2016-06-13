title: [转]调整虚拟机中ubuntu server屏幕分辨率
date: 2016-06-13 09:37:12
tags:
- ubuntu
- 分辨率
categories: 运维
---

转自：[http://blog.csdn.net/weilanxing/article/details/7664324](http://blog.csdn.net/weilanxing/article/details/7664324)。

VMware中的Ubuntu Server的控制台窗口有点儿小，使用起来不太方便，要调整控制台的窗口大小，需要修改屏幕的分辨率，修改方法如下：

1. 打开grub文件($vim /etc/default/grub), 修改参数GRUB_CMDLINE_LINUX的值,
GRUB_CMDLINE_LINUX="vga=0x317", 参数值参考下图：
```
| 640x480  800x600  1024x768 1280x1024
----|--------------------------------------
256 |  0x301    0x303    0x305    0x307
32k |  0x310    0x313    0x316    0x319
64k |  0x311    0x314    0x317    0x31A
16M |  0x312    0x315    0x318    0x31B
```
2. $sudo update-grub
3. $sudo reboot
