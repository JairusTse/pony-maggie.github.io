---
layout: post
title: 基于jekyll搭建自己的博客系统
category: tech
tags: [tech,jekyll,博客]
---

* content
{:toc}

## 安装步骤

jekyll的环境在本地电脑安装就可以了，不需要在部署博客的服务器上安装。看后面的流程就明白了，写博客的流程是在本地电脑书写markdown文件，然后编译生成静态web文件。最后是把这些文件上传到服务器上的web容器（nginx，tomcat等）运行即可。

我的环境是 win10

到这个网站下载ruby安装

https://rubyinstaller.org/downloads/

我下载的是rubyinstaller-2.3.3-x64的版本。一路点击下一步安装即可。中间有个地方需要勾选

```
add ruby executables to your path
```

继续在上面的网站下载，解压DevKit，我是解压到了

``` bash
C:\DevKit
```
打开命令行工具，进入该目录，执行

``` bash
ruby dk.rb init
```

完成后，生成了config.yml，定位到最后一行添加你自己ruby的安装路径，比如：

```bash
- C:/Ruby22-x64
```

然后还是在这个目录下，执行：

``` bash
ruby dk.rb review  # 审查（非必须）
ruby dk.rb install  # 安装
```

安装jekyll

``` bash
gem install jekyll
```

安装bundler

``` bash
gem install bundler -v 1.17.3
```

**我这里安装bundler的时候指定了版本，是因为之前用了最新版本出现了不兼容的问题**

如果你没有这样的问题，可以直接安装最新版本。

重新打开命令行，检查下是否安装成功：

``` bash
jekyll --version
```

基本的jekyll就安装完成了。

## 测试

``` bash
~$jekyll new myblog
~$ cd myblog/
~/myblog$ jekyll serve --host=0.0.0.0
```

然后通过http://127.0.0.1:4000/，就可以在浏览器看到页面效果了。

有些在网上下载的jekyll模板，使用了gemfile和bundle，这种需要进入的模板根目录下执行:

```
bundle install
```

如果这个命令执行很慢，可以参考下面这个文章更换ruby源。

https://ruby.taobao.org/

然后运行的时候，使用:

``` bash 
bundle exec jekyll serve
```

编译启动博客。

## 编写博客

编写博客概括来讲就是在_post目录下编写md格式的文章，写完后执行

``` bash 
bundle exec jekyll build
```

编译生成博客的web文件到_site目录下。具体的编写格式和规则，我就不啰嗦了，有官网教程。

https://www.jekyll.com.cn/docs/


### 使用别人的主题

自带的主题太过简陋，如果你前端比较牛逼可以自己开发一个主题，不过我更倾向于使用别人写好的主题。

**不要总想着自己造轮子，能用好轮子就行！**

我现在用的主题是：

https://github.com/DONGChuan/Yummy-Jekyll


另外，我这里总结了几个网上比较推崇的主题：

----------

仓库地址：https://github.com/DONGChuan/Yummy-Jekyll

效果：http://dongchuan.github.io/

------------

效果：https://blog.xinpapa.com/tag/

仓库地址：https://github.com/maoxiaoke/maoxiaoke.github.io

------------

效果：http://lanyon.getpoole.com/

仓库地址：https://github.com/poole/

-----------

效果：https://www.hifreud.com/

仓库地址：https://github.com/luoyan35714/LessOrMore

-----------

效果：https://www.panxw.com/

仓库地址：https://github.com/panxw/panxw.github.com

-----------

效果：http://liaokeyu.com/

仓库地址：https://github.com/panxw/panxw.github.com

-----------

更多可以去 

http://jekyllthemes.org/

查找自己喜欢的。


关于如何使用主题写博客，可以参考
[如何设置Jekyll主题](https://webdesign.tutsplus.com/zh-hans/tutorials/how-to-set-up-a-jekyll-theme--cms-26332)

## 购买域名和备案

你总得有个自己的域名吧，在中国大陆这个域名是需要备案的。能提供域名购买和备案的公司有挺多，我是用的阿里云的服务。这里不详述了，根据自己选择的服务商进行操作即可。

## 购买云服务器

我是购买的阿里云服务器，环境是ubuntu 16.04。记得把上一步备案好的域名解析到云服务器的公网IP上。


##  部署博客

web容器使用的nginx+tomcat方式。，nginx在前端，通过配置反向代理指向后端的tomcat。这个过程就不详述了，网上很多。

**nginx反向代理中配置的域名就是前面备案的域名。**


## 引入评论系统

第三方的评论系统有不少，我这里使用的是gitalk。

>Gitalk 是一个利用 Github API,基于 Github issue 和 Preact 开发的评论插件。在 gitalk 的评论框进行评论时，其实就是在对应的 issue 上提问题。

关于gitalk的配置，下面这篇文章说得比较详细：

https://www.jianshu.com/p/78c64d07124d

## 引入百度统计

我个人不喜欢在前台页面展示访问数量，百度统计正好适合我。

百度统计是后台统计，并没有再页面有任何展示，但是可以通过登录百度统计官网查看更详细的访问记录分析。

在博客中引入百度统计的方法参考官网教程配置即可，非常简单。


+ [百度统计](https://tongji.baidu.com)


## 引入Latex公式支持

把下面的 JS 调用代码以及配置插入到你 Jekyll 博客的 header 文件中。

``` script
<script type="text/x-mathjax-config">
  MathJax.Hub.Config({
    tex2jax: {
      skipTags: ['script', 'noscript', 'style', 'textarea', 'pre'],
      inlineMath: [['$','$']]
    }
  });
</script>
<script src='https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.5/latest.js?config=TeX-MML-AM_CHTML' async></script>
```

然后再markdown文件中，行中公式可以用如下方法表示：

```
    $ 数学公式 $
```

独立公式可以用如下方法表示：

```
   $$ 数学公式 $$
```

## issues

```
Deprecation: The 'gems' configuration option has been renamed to 'plugins'. Please update your config file accordingly.
```

>从 Jekyll 3.0 开始，jekyll-paginate 被移除了，因为与其他核心功能不能很好的协作。如果出现这种提示：

如果出现这个提示，解决办法：

安装插件
```
$ gem install ‘jekyll-paginate’
```
在 _config.yml 中增加：
```
plugins: [jekyll-paginate]
```

-----------

```
 Liquid Exception: Liquid error (D:/study/javascript/projects/machengyu.github.io/_includes/sidebar-popular-repo.html line 54): Cannot sort a null object. included in index.html
jekyll 3.8.5 | Error:  Liquid error (D:/study/javascript/projects/machengyu.github.io/_includes/sidebar-popular-repo.html line 54): Cannot sort a null object. included
```










