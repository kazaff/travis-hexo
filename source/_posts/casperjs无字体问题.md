title: casperjs无字体问题
date: 2017-09-20 09:37:12
tags:
- casperjs
- phantomjs
- centos

categories:
- nodejs
---

今天在服务器（centOS）上测试casperjs，突然发现截图出来的页面都是无字天书：

![](http://pic.yupoo.com/kazaff/GLdFmeia/medium.jpg)

 一开始以为是字体文件加载未完成导致的，后来发现原来是系统没有字体。。。shit

 查了[一下](https://stackoverflow.com/questions/15029002/phantomjs-screenshot-font-missing-boxes-rendered-instead)：

 ```
 yum install urw-fonts

 ```
 安装了这个字体，然后就搞定啦~~