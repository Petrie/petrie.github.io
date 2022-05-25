---
layout: post  
title: "Linux挂载ISO文件"  
description: "Linux挂载ISO文件"  
categories:
    - Linux
tags: [Linux,centos,mount,iso]    
date: 2012-03-06 10:26:12
---  

## Linux 下绑定ISO文件 ##  
在/mnt下创建/mnt/centos_iso文件夹
>mkdir /mnt/centos_iso
    
将/home/cksamba/share/下的centos.iso挂在到/mnt/centos_iso文件夹下
>mount /home/cksamba/share/centos.iso /mnt/centos_iso -t iso9660 -o ro,loop=/dev/loop0

或者
>mount  /home/cksamba/share/centos.iso /mnt/centos_iso -o loop -t iso9660
