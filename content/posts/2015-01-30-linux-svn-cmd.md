---
layout: post  
title: "Linux Svn Cmd"  
description: "Linux Svn常用命令"  
categories:
- svn 
tags: [Linux, svn]
date: 2015-01-30 10:26:12
---

```shell
svn diff [-r PREV] PATH
```

对比当前本地的工作拷贝文件(working copy)和任意版本A的差异  

```shell 
svn diff -r94239 
```

对比任意历史版本A和任意历史版本B的差异
```shell   
svn diff -r94239:94127
```