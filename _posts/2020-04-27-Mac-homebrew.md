---
layout: post
title:  "Mac 10.15.4安装Homebrew报错的问题"
author: mew151
image: assets/images/202004/clouds-forests-landscapes-nature-1920x1200-wallpaper.jpg
---
在Homebrew官网按照以下命令执行：

```shell
/bin/bash -c “$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install.sh)"
```

会报错：

```shell
curl: (7) Failed to connect to raw.githubusercontent.com port 443: Connection refused
```

根据[https://www.jianshu.com/p/b1de316fc3fc](https://www.jianshu.com/p/b1de316fc3fc)这篇文章的描述：
>  查询资料后发现苹果新系统安全提升，禁止了直接执行远程脚本。 把脚本文件下载到本地来执行就好。

而在安装完Homebrew之后，执行`brew install`时，会长时间卡在`Updating Homebrew...`

根据[https://learnku.com/articles/18908](https://learnku.com/articles/18908)这篇文章的描述：
>  按住control + c之后命令行会显示^C，就代表已经取消了Updating Homebrew操作。大概不到1秒钟之后就会去执行我们真正需要的安装操作了