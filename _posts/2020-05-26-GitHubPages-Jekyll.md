---
layout: post
title:  "使用GitHub Pages+Jekyll搭建个人博客网站"
author: mew151
image: assets/images/202005/in-nature-black-hair-choi-ji-hyang-1236x824-wallpaper.jpg
---

#### 创建博客
1、在你的GitHub上创建一个名为*username*.github.io的repository，*username*是你的GitHub账号名。

2、执行以下命令将这个repository拷贝到你本机：
```bash
git clone https://github.com/username/username.github.io
```
这时，你的个人博客就搭建好了，只不过它现在是一个空壳。:)

#### 安装Jekyll
1、替换源（国外的源实在是太慢了，科学上网也不行）
```bash
gem sources --add https://gems.ruby-china.com/ --remove https://rubygems.org/
bundle config mirror.https://rubygems.org https://gems.ruby-china.com
```
2、安装：
```bash
gem install bundler jekyll
bundle install
```

#### 为博客选择主题
你可以在[Jekyll Themes](http://jekyllthemes.org/)上选择你喜欢的主题。
以[Memoirs - Jekyll Bootstrap Theme](https://bootstrapstarter.com/bootstrap-templates/jekyll-theme-memoirs/)为例：
1. 进入到该主题的Homepage页面，将该主题下载到本机（可以使用git clone命令或直接下载zip包）。
2. 将下载的主题目录，除了.git目录以外的其他所有，移至你本机的*username*.github.io目录下。
3. 修改`_config.yml`文件中的`baseurl`属性，改为`''`。

#### 调试&发布
1、在本机*username*.github.io目录下，执行
```bash
bundle exec jekyll serve
```
访问[http://127.0.0.1:4000/](http://127.0.0.1:4000/)就可以在本机看到效果。

2、使用git push命令将*username*.github.io的修改推送至远端完成发布，
访问https://*username*.github.io即可看到效果。

---
参考资料：
- [https://pages.github.com/](https://pages.github.com/)
- [https://jekyllrb.com/](https://jekyllrb.com/)
- [https://gems.ruby-china.com/](https://gems.ruby-china.com/)
- [https://bootstrapstarter.com/bootstrap-templates/jekyll-theme-memoirs/](https://bootstrapstarter.com/bootstrap-templates/jekyll-theme-memoirs/)