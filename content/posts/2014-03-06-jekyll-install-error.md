---
layout: post  
title: "Jekyll Install Troubleshooting"  
description: "Jekyll Install Troubleshooting"  
categories:
    - Jekyll
tags: [Jekyll, Troubleshooting]  
date: 2014-03-06 10:26:12  
---

- 错误内容：
>ERROR:  Error installing jekyll:
>ERROR: Failed to build gem native extension.

高版本的ruby 导致的问题，修复命令：
>sudo apt-get install ruby1.9.1-dev
>gem install jekyll
