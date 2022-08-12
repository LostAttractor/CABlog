---
layout: "posts"
title: "更新gdk-pixbuf2后无法正常加载图片"
date: 2022-08-12T15:00:00+08:00
---

在我今天习惯性的使用pacman -Syu对我的archlinux进行更新并重启后，我桌面壁纸变成了纯黑  
对软件包进行逐一降级后发现gdk-pixbuf2的2.42.9-1更新似乎有bug，会导致在我的环境下gtk程序和gnome无法正常显示图片  
解决方法也很简单，使用pacman -U进行降级就好了

![updategdkpixbuf2.avif](/img/diray/updategdkpixbuf2.avif)