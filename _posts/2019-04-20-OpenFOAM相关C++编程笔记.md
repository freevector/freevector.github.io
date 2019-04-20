---
bg: "think.jpg"
layout: post
title:  "C++学习笔记-多态性"
crawlertitle: "Learning"
summary: "Something to share"
categories: posts
tags: ['C++']
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
  
### C++学习笔记-多态性
#### OpenFOAM相关

在openfoam中，多态应用很广，例如以下模型都用到了多态：
+ Turbulence models
+ Boundary conditions
+ functionObjects

如果我们想创建一个新的湍流模型，只需要从基类继承
c++的多态性用一句话概括就是:在基类的函数前加上virtual关键字，在派生类中重写该函数，运行时将会根据对象的实际类型来调用相应的函数，如果对象类型是派生类，就调用派生类的函数，如果对象类型是基类，就调用基类的函数。

虚函数是在基类中定义的，目的是不确定它的派生类的具体行为，例如:定义一个基类:class Animal //动物，它的函数为breathe()；再定义一个类class Fish //鱼。它的函数也为breathe()；再定义一个类class Sheep //羊，它的函数也为breathe()

将Fish，Sheep定义成Animal的派生类，然而Fish与Sheep的breathe不一样，一个是在水中通过水来呼吸，一个是直接呼吸，所以基类不能确定该如何定义breathe，所以在基类中只定义了一个**virtual breathe，它是一个空的虚函数**，具体的函数在子类中分别定义，程序一般运行时，找到类，如果它有基类，再找到它的基类，最后运行的是基类中的函数，这时，它在基类中找到的是virtual标识的函数，它就会再回到子类中找同名函数，派生类也叫子类，基类也叫父类，这就是虚函数的产生，和类的多态性的体现。
这里的多态性是指**类的多态性**。派生类对象的地址可以赋值给基类指针
```

    #include <iostream>
    using namespace std;
    class A
    {
    public:
        virtual void Print() { cout << "A::Print" << endl; }
    };
    class B : public A
    {
    public:
        virtual void Print() { cout << "B::Print" << endl; }
    };
    class D : public A
    {
    public:
        virtual void Print() { cout << "D::Print" << endl; }
    };
    class E : public B
    {
        virtual void Print() { cout << "E::Print" << endl; }
    };
    int main()
    {
        A  a; B b; D d; E e;
        A *pa = &a;  B *pb = &b;
        pa->Print();    //多态， a.Print()被调用，输出：A::Print
        pa = pb;        //基类指针pa指向派生类对象b
        pa->Print();  //b.Print()被调用，输出：B::Print
        pa = &d;       //基类指针pa指向派生类对象d
        pa->Print();  //多态， d. Print ()被调用,输出：D::Print
        pa = &e;       //基类指针pa指向派生类对象d
        pa->Print();  //多态， e.Print () 被调用,输出：E::Print
        return 0;
    }
    ```

每个类都有同名、同参数表的虚函数 Print（每个 Print 函数声明时都加了 virtual 关键字）。根据多态的规则，对于语句“pa->Print()”，由于 Print 是虚函数，尽管 pa 是基类 A 的指针，编译时也不能确定调用的是哪个类的 Print 函数。当程序运行到该语句时，pa 指向的是哪个类的对象，调用的就是哪个类的 Print 函数。
 例如，程序执行到第 26 行时，pa 指向的是基类对象 a，因此调用的就是类 A 的 Print 成员函数；执行到第 28 行时，pa 指向的是类 B 的对象，因此调用的是类 B 的 Print 成员函数；第 30 行也是如此；类 E 是类 A 的间接派生类，因此，执行到第 32 行时，多态规则仍然适用，此时 pa 指向派生类 E 的对象，故调用的是类 E 的 Print 成员函数。



 <span id="busuanzi_container_page_pv">
  本文总阅读量<span id="busuanzi_value_page_pv"></span>次
</span>

