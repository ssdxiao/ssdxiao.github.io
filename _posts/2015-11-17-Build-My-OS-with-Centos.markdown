---

layout: post

title:  通过原始Centos ISO来定制自己的ISO

date:   2015-11-17 17:10:00
categories: linux
tags: 
  - linux
---

通过Centos原始镜像可以裁剪或者增加rpm包制作自己的镜像



下载centos7的镜像 mount 镜像到本地 

{%highlight bash%}
mount CentOS-7.0-1406-x86_64-DVD .iso /mnt
{%endhighlight%}

因为镜像本身是只读的，所以需要将镜像cp出来 

{%highlight bash%}
cp /mnt/* /home/myiso/*
cp /mnt/.treeinfo /home/myiso/
cp /mnt/.discinfo /home/mysio
{%endhighlight%}

修改rpm包并createrepo 
在repodata目录下会看到一个xml文件，该文件主要定制了安装模式，以及每一种模式下需要安装的rpm包。如果要定

制自己的ISO，主要就是在这里进行修改。


需要复制原来的2bc0054a9f0f4cd3d2806d983edbe3d0dfc484d9f275d12be79eb67a040ba942-c7-x86_64-comps.xml 到

c7-x86_64-comps.xml 然后创建仓库 

{%highlight bash%}
rm repodata/*comps.xml.gz -rf
mv repodata/*c7-x86_64-comps.xml repodata/c7-x86_64-comps.xml
createrepo -g repodata/c7-x86_64-comps.xml  ./
{%endhighlight%}
修改.treeinfo中的对应校验值使用sha256sum 

{%highlight bash%}
sha256sum repodata/repomd.xml
{%endhighlight%}
打包 
{%highlight bash%}
mkisofs -J -R -T -v -V "CentOS 7 x86_64" -boot-info-table  -no-emul-boot -boot-load-size 4 -b isolinux/isolinux.bin -c isolinux/boot.cat -o  ../CentOS-DevOps.iso ./
{%endhighlight%}

添加md5的校验 

{%highlight bash%}
implantisomd5 CentOS-DevOps.iso
{%endhighlight%}
