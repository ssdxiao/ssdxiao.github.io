---

layout: post

title:  windows-docker介绍
date:   2016-8-5 14:10:00
categories: 容器
tags: 
  - 容器
  - docker 
---

背景：
---
windows上支持docker的新闻已经不算陌生了。但是究竟现在发展到什么地步了呢？
心血来潮，开始在网上寻找在windows上使用docker的方式。


part 1 docker官方给的docker for windows
---

[Getting Started with Docker for Windows]介绍了如何在windows上使用docker。
首先环境需要64bit Windows 10 Pro，然后需要开启HyperV服务。
然后下载安装包就可以了。
安装过程还是相当顺利。一路下一步就OK。
然后就可以开始体验我们的windows下的容器服务。


官方给的例子使用了ubuntu镜像。
这里一个大大的疑问涌上心头, windows容器服务已经可以直接跑linux的容器镜像了？
带着好奇心，启动了ubuntu镜像。

{%highlight bash%}
docker run -it ubuntu bash
{% endhighlight %}

进入镜像不免想看看，到底是什么情况。uname -a 来查看下内核版本。
4.4.5的内核出现在了眼前。难道其实windows上跑的容器，仅仅是跑在一个虚拟机里面。

打开HyperV的管理器，可以看到，的确启动了一台虚拟机。
该虚拟机使用了MobyLinuxVM.vhdx文件作为了数据盘，用来存储docker image文件。mobylinux.iso作为启动盘
来加载linux系统。

好吧。这样原理就明朗了。使用HyperV虚拟机运行了一个定制版本的linux，
然后windows装了docker来跟这个定制版的linux进行通信，达到了对容器使用的目的。使用的镜像就是linux的镜像。

原理明朗了，但是有一些莫名的失望，windows上的容器是这么使用的吗？那windows原生程序怎么跑呢？

使用
{%highlight bash%}
docker search microsoft
{% endhighlight %}

有些惊喜的发现
{%highlight bash%}
NAME                                          DESCRIPTION                                     STARS     OFFICIAL   AUTOM
ATED
microsoft/aspnet                              ASP.NET is an open source server-side Web ...   464                  [OK]
microsoft/dotnet                              Official images for working with .NET Core...   207                  [OK]
mono                                          Mono is an open source implementation of M...   172       [OK]
microsoft/azure-cli                           Docker image for Microsoft Azure Command L...   61                   [OK]
microsoft/iis                                 Internet Information Services (IIS) instal...   25
microsoft/mssql-server-2014-express-windows   Microsoft SQL Server 2014 Express installe...   23
microsoft/nanoserver                          Nano Server base OS image for Windows cont...   11
microsoft/windowsservercore                   Windows Server Core base OS image for Wind...   8
microsoft/dotnet-preview                      Preview bits for microsoft/dotnet image         5                    [OK]
microsoft/oms                                 Monitor your containers using the Operatio...   5                    [OK]
microsoft/applicationinsights                 Application Insights for Docker helps you ...   3                    [OK]
microsoft/sample-nginx                        Nginx installed in Windows Server Core and...   2
microsoft/sample-node                         Node installed in a Nano Server based cont...   2
microsoft/sample-redis                        Redis installed in Windows Server Core and...   2
microsoft/dotnet35                                                                            2
microsoft/sqlite                              SQLite installed in a Windows Server Core ...   1
microsoft/sample-httpd                        Apache httpd installed in Windows Server C...   1
microsoft/sample-mongodb                                                                      1
microsoft/sample-mysql                        MySQL installed in Windows Server Core and...   1
microsoft/sample-dotnet                       .NET Core running in a Nano Server container    1
microsoft/sample-golang                       Go Programming Language installed in Windo...   0
microsoft/sample-python                       Python installed in Windows Server Core an...   0
microsoft/sample-ruby                         Ruby installed in a Windows Server Core ba...   0
microsoft/dotnet-nightly                      Preview bits of the .NET Core CLI               0                    [OK]
microsoft/sample-rails                        Ruby on Rails installed in Windows Server ...   0
{% endhighlight %}

已经有了microsoft的镜像了，那么尝试下载了一个microsoft/iis的镜像。在执行

{%highlight bash%}
docker run -it microsoft/iis cmd
{% endhighlight %}

但是报错误。提示找不到$PATH变量。

到这里，基本可以确定，这个windows镜像不是这样使用的。那么windows上的容器服务该怎么用呢。

part 2
---

峰回路转，google一下就能看见，在MSDN网站有专门介绍windows docker服务如何体验的。

根据[windows上docker安装]提示，就可以真正的体验windows的容器服务。

首先说明下，截止现在windows的容器服务还是没有正式发布的。但是作为开发人员已经可以开始进行一些工作了。

环境要求 Windows 10 预览版，需要在win10中进行更新到一个开发人员版。
然后按照文章中介绍的进行安装就可以正常启动容器。


这里说一下，注意的地方：


- 从windows docker介绍的英文版中下载的docker是1.2.0版的。这个版本存在bug，在windows上会出错。
中文版介绍页面直接下载1.3.0的是可以正常使用的。
- windows的容器需要开启HyperV服务。
- windows存在两种容器，一种是windows上的，在win2016server版中使用的是这种。一种是借助HyperV的，在win10使用的就是借助HyperV的
- 在win10中使用的容器使用了nanoserver这个镜像，这个镜像就是一个裁剪过的微缩版windows。而win2016server上直接使用的是windowsservercore镜像。
- 在win10的容器，其实还是使用HyperV的虚拟机启动了nanoserver这个镜像，然后再镜像里面运行windows程序。每一个docker容器都是一个虚拟机。
- 根据文档，在winserver中启动的容器应该类似于linux的容器，真正公用了宿主机的内核。

小结：
---
从windows提供容器服务可以看到，docker的本质其实应该是镜像+资源隔离。和使用容器或者虚拟机本身来说关系并不那么密切。
在windows下，为了能够启动linux的程序，还是需要借助虚拟机来做操作系统隔离。而windows本身也存在有个人版和服务器版。
由于这2个版本内核形式的不同，windows为了让个人版能够使用容器服务目前也采取了虚拟机来启动一个微缩的内核。
这一思路也许可以在linux上沿用。在linux上启动nanoserver的虚拟机，然后跑windows程序。从这个角度看，摒弃容器本身带来的性能
优越性，将其封装在虚拟机层面来提供服务可以使得容器服务真正的操作系统无关。而这对现在企业中大量 windows服务进行容器化改造也有着深远的意义。



[Getting Started with Docker for Windows]: https://docs.docker.com/docker-for-windows/

[windows上docker安装]: https://msdn.microsoft.com/virtualization/windowscontainers/quick_start/quick_start_windows_10



