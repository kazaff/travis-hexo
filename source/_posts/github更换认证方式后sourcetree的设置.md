title: github更换认证方式后sourcetree的设置
date: 2020-11-28 09:37:12
tags: 
- github
- sourcetree
  
categories: 工具
---

个人非常喜欢sourcetree这款git客户端工具，比较喜欢它的设计理念，即将所有功能都集成在同一个UI中，而不是散落在不同的菜单或文件夹里。
不过之前一直用github的https协议来作为项目的git地址，sourcetree似乎对https协议的项目默认使用的认证授权方式就是账号密码。
但是随着github近期调整了认证方式，不再允许第三方工具基于账号密码来访问和管理项目了，所以我在sourcetree无法推送更新到github的代码仓库了。。。。
这就尴尬了。


一开始认为github推荐的OAuth方案，sourcetree应该也提供了对接吧。可不巧的是并没有，这就尴尬了。
不过我们还可以使用ssh协议+证书的方式来打通sourcetree和github，说干就干！！

首先，打开你的sourcetree，再顶部主菜单中选择 “Tools -> Create or Import SSH Keys”，在弹出的窗口中点击“Generate”，根据提示再窗口中央的空白区域不停的摩擦鼠标直至生成完毕，然后我们把创建的“public key”的字符串拷贝好，并点击“Save private key”将私钥文件保存在文件中。

接下来，登录你的github的web后台，点击右上角你的头像，选择 “setting -> SSH and GPG keys”，在跳转后的页面，左上角点击“New SSH key”，将上一步拷贝的公钥字符串黏贴在页面的“Key”输入框中，“Title”填写一个你觉得合适的名字即可。

最后，再回到sourcetree的界面， 点击 “Tools -> Options” 打开sourcetree的选项菜单，然后在 “General” 选项卡中找到“SSH Client Configuration”设置块，其中的 “SSH Key”项我们就设置成前面我们私钥文件即可。

最后，在github中获取clone链接的时候，记得切换成SSH协议，用该协议的项目地址作为sourcetree使用的仓库地址，就完成了所有的配置哦。

祝福好运~~