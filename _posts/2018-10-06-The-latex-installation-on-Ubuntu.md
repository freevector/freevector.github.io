---
bg: "think.jpg"
layout: post
title:  "The installation of latex on ubuntu"
crawlertitle: "The installation of latex"
summary: "The installation of latex on ubuntu"
categories: posts
tags: ['Latex']
author: Vector CHOW
---
### Latex在ubuntu系统的安装
**Latex** 在科技论文排版方面优势明显，很多期刊都有相应的模板。Ubuntu系统下可以通过如下几个步骤安装latex。  

1. 安装latex，本文安装的是分发版Tex Live  
```vim
sudo apt-get install texlive-full
```
2. 第二步，安装中文字体包。字体包中包含bsmi，bkai，gkai，gbsn四种中文字体。bsmi和bkai是Big5编码的宋体和楷体字；后两者gkai和gbsn分别处理简体中文楷体字和宋体字。
```vim
sudo apt-get install latex-cjk-all
```
3. 安装文本编辑器，本文选择安装texmaker
```vim
sudo apt-get install texmaker
```
4. 在 Ubuntu 下执行下面命令可以打开 Texmaker 编辑器：
```vim
texmaker
```
5. 现在让我们用 Texmaker 创建一个简单的文档，点击 File -> New 然后在新文档中插入如下内容：
```vim
\documentclass{article} \begin{document} Hello oschina! \end{document} 
```