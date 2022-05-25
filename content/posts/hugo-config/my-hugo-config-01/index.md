---
title: "Hugo配置中级技巧(1)：评论、网站统计、日期、字数、时长"
date: 2022-05-23T20:44:24+08:00
tags :
    - "hugo"
    - "tutorial"
categories:
    - "hugo"
isCJKLanguage: true
featured_image: "./images/hugo_logo.png"
---

## 一、支持评论
先看配置改动 **[9e9acc](https://github.com/Petrie/petrie.github.io/commit/9e9acc55a16677cc603b1610a8ce63dabfda3e9e)**  
Disqus 的交互设计太傻X了。     
如何正确获取你需要配置的内容，见下面的导引：  
[首页](https://disqus.com/)->Get Start -> I want to install Disqus on my site  -> Website Name。   
这个Website Name 就是你需要再配置文件填入的内容   

参考资料，Hugo官方文档: [点击](https://gohugo.io/content-management/comments/)   

<!--more-->
## 二、配置Google Analytics  

先看配置改动 **[894ac5](https://github.com/Petrie/petrie.github.io/commit/894ac5007eb9aefb703a39086198f7f765c67b3d)**
如何获取配置内容，见如下操作步骤：
进入[配置首页](https://analytics.google.com/analytics/web/?authuser=0#/a101431760p316225154/admin")

**第一步：点击"Create Property"**  
{{< figure src="./images/img.png" height="500" link="./images/img.png">}}

**第二步：填入名称，选择时区和货币** 
{{< figure src="./images/img_1.png" height="500" link="./images/img_1.png">}}

**第三部：选择行业分类**
{{< figure src="./images/img_2.png" height="500" link="./images/img_2.png">}}

**第四部：选择平台：Web**
{{< figure src="./images/img_3.png" height="500" link="./images/img_3.png">}}

**第五步：填入网站地址和流名称**
{{< figure src="./images/img_4.png" height="500" link="./images/img_4.png">}}

**第六部：获取ID**   
右上角的`MEASUREMENT ID`即为需要填入配置文件的内容 
{{< figure src="./images/img_5.png" height="500" link="./images/img_5.png">}}
按以上流程配置完后，刷新页面  
进入 Google Analytics [首页](https://analytics.google.com) ,查看 View realtime部分不为0则说明接入成功了

## 三、配置日期格式、文章字数、阅读时长  

**1、日期格式配置：**  
```toml
#file:config.toml
[params]
    date_format = "2006年01月2日" #日期格式调整的配置
```
**2、字数、时长配置：**  
**2.1 先更新配置文件**  
```toml
#file:config.toml
[params]
    show_reading_time = true #支持显示文章字数和阅读市场的配置
```
此时文章标题下方已经可以显示了，但是还是英文，如何调整问中文呢？  


**2.2 关键字汉化**  
首先，将配置文件：`themes/ananke/i18n/zh.toml` 复制到 `i18n/zh.toml  `  
```
├── i18n
│   └── zh.toml
└── themes
    └── ananke
        ├── i18n
        │   ├── en.toml
        │   └── zh.toml #复制它到 i18n/zh.toml
```
并且更新配置文件，此时才会应用`i18n/zh.toml`的配置  
```toml
#file:config.toml
DefaultContentLanguage = "zh"
```
但这个时候仍然是中文，这是因为`i18n/zh.toml`配置没有覆盖全  
在其尾巴添加如下配置：  
```toml
#file:config.toml
[readingTime]
other = "{{ .Count }} 分钟"

[wordCount]
other = "{{ .Count }} 字"
```

此时可以看到已经是中文了 ，但是还有问题，此时的字数统计是不准确的。

```yaml
#file:content:/posts/example.md
isCJKLanguage: true
```

完