---
bg: "think.jpg"
layout: post
title:  "About Helmholtz Resonance"
crawlertitle: "resonance"
summary: "noise"
categories: posts
tags: ['sound-science']
author: Vector CHOW
---
<script type="text/x-mathjax-config">
    MathJax.Hub.Config({
      tex2jax: {
        skipTags: ['script', 'noscript', 'style', 'textarea', 'pre'],
        inlineMath: [['$','$']]
      }
    });
  </script>
  <script src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML" type="text/javascript"></script>
### 亥姆霍兹共振器
**亥姆霍兹共振器**是一个带开口（孔/颈）结构的气体容器（通常为空气），靠近开口处的气体做类似于弹簧的往复运动。一个常见的例子就是当你略带一点角度吹瓶口的时候，会发出特定频率的尖啸声。  
![]({{ site.images }}/posts/helm_resonator.jpg)
一些小口哨其实就是**亥姆霍兹共振器**。吉他共鸣腔也起到类似于**亥姆霍兹共振器**的作用。另外还有陶笛(*Ocarina*)也是一种**亥姆霍兹共振器**，只不过结构复杂一点，为了实现不同的音节，使用了多个孔而已。另外，对于音响系统通常也利用亥姆霍兹共振原理实现低音强化。接下来我们简单推导一下**亥姆霍兹共振器**的频率公式。  
![]({{ site.images }}/posts/Helmholtz1.GIF)
首先假设该共振器发出的波长大于共振器本身的尺寸，后面我们会根据计算结果对该假设进行检验；在上述假设的条件下，我们可以忽略容器内的压力波动。
![]({{ site.images }}/posts/Helmholtz2.GIF)
假设容器接管长L，气体密度为$\rho$ ，根据热力学方程有:  
$$PV^\gamma=Const \Rightarrow dP/P=-\gamma dV/V$$  
又因为$dV=Sdx $，根据牛顿第二定理：  

$$a=F/m$$  

$$F=dP*S=-\gamma PSdV/V$$  

$$m=\rho SL$$  

$$\Rightarrow a= -\gamma PdV/\rho LV= -\gamma PSdx/\rho LV$$  

往复力同位移成正比且反向，显然类似于弹簧力，有如下周期  

$$f=\sqrt(\gamma PSdx/\rho LV)/2\pi=c\sqrt(S/VL)/2\pi$$  
![]({{ site.images }}/posts/HelmholtzResonator.jpg)
> 版权归[原文所有](http://newt.phys.unsw.edu.au/jw/Helmholtz.html) ，本文仅为中文翻译。