---
layout: post
title:  "搭建jekyll"
date:   2015-08-26 21:13:31
categories: jekyll
---

开始有了好好写点文章的想法，本人一直比较懒，所以以前的文章也就马马虎虎的。
既然打算开始就准备坚持下去。

第一次写就写下如何使用jekyll来搭建吧。

jekyll总的来说还是比较简单的，网上的教程比较多了
个人推荐[blog1]介绍了jekyll的原理。


当然如果要自己搭建还是要使用还是最好使用已有模板来进行开发。
当然首先需要有一个本地的jekyll环境。可以参考[blog2]。


简单总结下我的搭建流程：
由于使用的时mac，相对比较简单。我的mac上默认安装的就有ruby，所以直接安装
jesykll就可以了
执行以下命令

{%highlight bash%}
$ sudo gem sources --remove https://rubygems.org/
$ sudo gem sources -a https://ruby.taobao.org/

$ sudo gem install jekyll
{% endhighlight %}
就这么简单几个命令就可以开始体验jekyll了。

如果要使用jekyll自带模板就使用下面的命令
{%highlight bash%}
$ jekyll blog 
$ cd blog 
$ jekyll server
{% endhighlight %}

这样访问本地http://127.0.0.1:4000就可以看见自己的blog了。



附录：如何使用[代码高亮]


[blog1]: http://www.ruanyifeng.com/blog/2012/08/blogging_with_jekyll.html
[blog2]: http://blog.csdn.net/chziroy/article/details/38837715
[代码高亮]: http://pygments.org/docs/lexers/

