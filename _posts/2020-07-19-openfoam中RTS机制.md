---
bg: "think.jpg"
layout: post
title:  "openfoam学习笔记-9"
crawlertitle: "Learning"
summary: "OpenFoam learning notes"
categories: posts
tags: ['OpenFOAM']
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
  
### 用 C++11 实现迷你版本的运行时选择<转>
本文用 C++11 标准实现了一个类似 OpenFOAM 运行时选择机制的示例。通过这个示例可以更加容易直观地了解运行时选择内部的实现原理。

#### 一个简单的例子
考虑用经典的简单工厂模式实现的以下代码（mini-rts_v1.cpp），这段代码实现了用同一个函数接口，传递不同的字符串，得到不同的派生类实例对象。

设计一个时间离散格式的基类，定义如下：
```
class ddtScheme
{
public:
    ddtScheme() {}
};
```
> 实际上 ddtScheme 为类模板，情况更为复杂，为了便于演示这里做了简化。

设计一个 Euler 格式的派生类，定义如下：
```
class EulerDdtScheme
:
    public ddtScheme
{
public:
    EulerDdtScheme() { std::cout << "Construct Euler scheme." << std::endl; }
};
```
设计一个 backward 格式的派生类，定义如下：
```
class backwardDdtScheme
:
    public ddtScheme
{
public:
    backwardDdtScheme() { std::cout << "Construct backward scheme." << std::endl; }
};
```
设计一个工厂类，包含工厂方法，用于创建 ddtScheme 对象，返回的对象用智能指针 shared_ptr 表示：
```
class ddtSchemeFactory
{
public:
    ddtSchemeFactory() {}

    static std::shared_ptr<ddtScheme> createDdtScheme(const std::string& type)
    {
        if (type == "Euler")
        {
            return std::shared_ptr<ddtScheme>(new EulerDdtScheme);
        }
        else if (type == "backward")
        {
            return std::shared_ptr<ddtScheme>(new backwardDdtScheme);
        }
        else
        {
            std::cout << "Unknown type: " << type << std::endl;
            return nullptr;
        }
    }
};
```
> 上面的 shared_ptr 是 C++11 标准中的智能指针，OpenFOAM 中的 autoPtr 和它功能类似。

main 函数代码如下：

```
int main(int argc, char *argv[])
{
    std::shared_ptr<ddtScheme> euler = ddtSchemeFactory::createDdtScheme("Euler");
    std::shared_ptr<ddtScheme> banana = ddtSchemeFactory::createDdtScheme("banana");

    return 0;
}
```

编译运行后结果如下：
```
$ g++ mini-rts_v1.cpp -o mini-rts_v1
$ ./mini-rts_v1
Construct Euler scheme.
Unknown type: banana
```
维护成本问题
简单工厂模式维护不方便。若要增加一个新的时间离散格式，需要修改两个地方：一是增加派生类的声明和定义；二是修改工厂方法，增加相应的判断。

例如我们要增加 steadyState 时间格式（mini-rts_v2.cpp），首先是增加相应的类：
```
class steadyStateDdtScheme
:
    public ddtScheme
{
public:
    steadyStateDdtScheme() { std::cout << "Construct steadyState scheme." << std::endl; }
};
```
然后是修改工厂方法，增加相应的判断：
```
std::shared_ptr<ddtScheme> createInstance(const std::string& type)
{
    if (type == "Euler")
    {
        return std::shared_ptr<ddtScheme>(new EulerDdtScheme);
    }
    else if (type == "backward")
    {
        return std::shared_ptr<ddtScheme>(new backwardDdtScheme);
    }
    else if (type == "steadyState")
    {
        return std::shared_ptr<ddtScheme>(new steadyStateDdtScheme);
    }
    else
    {
        std::cout << "Unknown type: " << type << std::endl;
        return nullptr;
    }
}
```
main 函数如下：
```
int main(int argc, char *argv[])
{
    std::shared_ptr<ddtScheme> euler = ddtSchemeFactory::createDdtScheme("Euler");
    std::shared_ptr<ddtScheme> steadystate = ddtSchemeFactory::createDdtScheme("steadyState");
    std::shared_ptr<ddtScheme> banana = ddtSchemeFactory::createDdtScheme("banana");

    return 0;
}
```
编译运行后结果如下：
```
$ g++ mini-rts_v2.cpp -o mini-rts_v2
$ ./mini-rts_v2
Construct Euler scheme.
Construct steadyState scheme.
Unknown type: banana
```
改进版本
为了降低维护成本，我们对设计进行改进（mini-rts_v3.cpp）。在 ddtScheme 中引入一个 std::unordered_map 对象，用来保存一个字符串到函数的映射关系。通过字符串查找对应函数，利用该函数构造相应的派生类对象。

在 ddtScheme 基类中增加以下声明：
```
  // 为构造函数定义类型别名
    typedef std::shared_ptr<ddtScheme> (*constructorPtr)();

    // 为 map 定义类型别名
    typedef std::unordered_map<std::string, constructorPtr> constructorMap;

    // 声明 map 对象指针
    static constructorMap *constructorMapPtr_;
 ```
声明指针后还需要在类外部定义指针：
```
ddtScheme::constructorMap *ddtScheme::constructorMapPtr_ = NULL;
```
设计一个静态函数，用于创建指针对象：
```
 // 创建 map 对象
    static void constructConstructorMaps()
    {
        static bool constructed = false;
        if (!constructed)
        {
            constructed = true;
            constructorMapPtr_ = new constructorMap;
        }
    }
```
设计一个函数模板，用于构造派生类对象：
```
template<typename ddtSchemeType>
    static std::shared_ptr<ddtScheme> ddtSchemeNew()
    {
        return std::shared_ptr<ddtScheme>(new ddtSchemeType);
    }
```
以派生类作为模板参数代入上面的函数模板，可以得到返回不同派生类对象的函数，比如 ddtSchemeNew<EulerDdtScheme>() 将返回一个 EulerDdtScheme 对象。

重新设计工厂类 ddtSchemeFactory 中的工厂方法，去掉繁琐的 if、else if 结构，改用 map 的 find 方法实现查找：
```
static std::shared_ptr<ddtScheme> createDdtScheme(const std::string& type)
    {
        ddtScheme::constructorMap::iterator it = ddtScheme::constructorMapPtr_->find(type);
        if (it != ddtScheme::constructorMapPtr_->end())
        {
            return it->second();
        }
        else
        {
            std::cout << "Unknown type: " << type << std::endl;
            return nullptr;
        }
    }
```
这样便完成了初步改造，但是目前 map 对象中没有内容。需要设计一个函数，用于往 map 对象中添加条目：
```
 template<typename ddtSchemeType>
    static void addToConstructorMaps(const std::string& lookup)
    {
        ddtScheme::constructConstructorMaps();
        ddtScheme::constructorMapPtr_->insert(ddtScheme::constructorMap::value_type(lookup, ddtSchemeNew<ddtSchemeType>));
    }
```
修改 main 函数，在执行工厂方法前，先调用相关函数初始化 map 对象：
```
int main(int argc, char *argv[])
{
    // 初始化 map 对象
    ddtScheme::constructConstructorMaps();

    // 往 map 对象中添加条目
    ddtScheme::addToConstructorMaps<EulerDdtScheme>("Euler");
    ddtScheme::addToConstructorMaps<backwardDdtScheme>("backward");
    ddtScheme::addToConstructorMaps<steadyStateDdtScheme>("steadyState");

    std::shared_ptr<ddtScheme> euler = ddtSchemeFactory::createDdtScheme("Euler");
    std::shared_ptr<ddtScheme> steadystate = ddtSchemeFactory::createDdtScheme("steadyState");
    std::shared_ptr<ddtScheme> banana = ddtSchemeFactory::createDdtScheme("banana");

    return 0;
}
```
编译运行后结果如下：

```
$ g++ mini-rts_v3.cpp -o mini-rts_v3
$ ./mini-rts_v3
Construct Euler scheme.
Construct steadyState scheme.
Unknown type: banana
```
进一步改进
虽然对简单工厂进行改进，避免了工厂方法中繁琐的 if、else if 判断。但是改进版本每次添加条目都要手动执行相关函数初始化 map，这样依然不方便。是否可以进一步改进，实现初始化 map 相关函数的自动执行？答案是肯定的，可以利用静态变量的特性实现。

这里要介绍静态变量的特点。C++ 编译器在编译代码时，所有静态变量在编译完成后就已经完成初始化并保存在二进制文件中。因此在执行 main 函数前所有静态变量已经初始化完成。可以利用这一特性实现我们想要的功能，接下来介绍具体思路。

对于初始化 map 对象功能，可以设计一个类，并使得该类的构造函数执行初始化 map 对象的函数。然后定义一个该类的静态对象。

定义 addConstructorToMap 类模板，将之前的 addConstructorToMaps 函数作为该类模板的构造函数，同时将 ddtSchemeNew 函数移动到这个类模板中 ：
```
  // 用于实现初始化 map 对象的类模板，定义该类的静态变量以实现自动添加条目至 map 对象
    template<typename ddtSchemeType>
    class addConstructorToMap
    {
    public:
        static std::shared_ptr<ddtScheme> ddtSchemeNew()
        {
            return std::shared_ptr<ddtScheme>(new ddtSchemeType);
        }

        addConstructorToMap(const std::string& lookup = ddtSchemeType::typeName)
        {
            ddtScheme::constructConstructorMaps();
            ddtScheme::constructorMapPtr_->insert(ddtScheme::constructorMap::value_type(lookup, ddtSchemeNew));
        }
    };
```
这个类模板需要用派生类进行模板特化，因此每个派生类都需要有 typeName 这个成员变量。下面为类加上这个变量：
```
class EulerDdtScheme
:
    public ddtScheme
{
public:
    static const std::string typeName;
    EulerDdtScheme() { std::cout << "Construct Euler scheme." << std::endl; }
};


class backwardDdtScheme
:
    public ddtScheme
{
public:
    static const std::string typeName;
    backwardDdtScheme() { std::cout << "Construct backward scheme." << std::endl; }
};


class steadyStateDdtScheme
:
    public ddtScheme
{
public:
    static const std::string typeName;
    steadyStateDdtScheme() { std::cout << "Construct steadyState scheme." << std::endl; }
};
```
在类外部定义 typeName，同时用派生类进行模板特化：
```
// 定义 typeName
const std::string EulerDdtScheme::typeName = "Euler";
const std::string backwardDdtScheme::typeName = "backward";
const std::string steadyStateDdtScheme::typeName = "steadyState";

// 用模板特化后的类定义静态变量
ddtScheme::addConstructorToMap<EulerDdtScheme> addEulerConstructorToTable_;
ddtScheme::addConstructorToMap<backwardDdtScheme> addbackwardConstructorToTable_;
ddtScheme::addConstructorToMap<steadyStateDdtScheme> addsteadyStateConstructorToTable_;
```

删除工厂类，将工厂方法放到基类中，改名为 New：
```
class ddtScheme
{
public:
    ddtScheme() {}

    static std::shared_ptr<ddtScheme> New(const std::string& type)
    {
        ddtScheme::constructorMap::iterator it = ddtScheme::constructorMapPtr_->find(type);
        if (it != ddtScheme::constructorMapPtr_->end())
        {
            return it->second();
        }
        else
        {
            std::cout << "Unknown type: " << type << std::endl;
            return nullptr;
        }
    }

    // ...
```
最后修改 main 函数，删除初始化 map 对象的相关函数：
```
int main(int argc, char *argv[])
{
    std::shared_ptr<ddtScheme> euler = ddtScheme::New("Euler");
    std::shared_ptr<ddtScheme> steadystate = ddtScheme::New("steadyState");
    std::shared_ptr<ddtScheme> banana = ddtScheme::New("banana");

    return 0;
}
```
编译运行后结果如下：
```
$ g++ mini-rts_v4.cpp -o mini-rts_v4
$ ./mini-rts_v4
Construct Euler scheme.
Construct steadyState scheme.
Unknown type: banana
```
到这里已经比较接近 OpenFOAM 中运行时选择的实现，接下来的工作就是将静态变量的声明用宏定义替换。

涉及到的宏及其大致对应的功能如下：

TypeName：声明静态成员变量 typeName

defineTypeNameAndDebug：定义静态成员变量 typeName

declareRunTimeSelectionTable：声明 map 对象，定义 addConstructorToMap 类模板

defineRunTimeSelectionTable：定义 map 对象，定义 constructConstructorMaps

addToRunTimeSelectionTable：定义模板特化后的 addConstructorToMap 对象，往 map 中添加条目

附录
附上本文涉及到的所有源文件的完整代码，需要支持 C++11 的编译器才能成功编译。

##### mini-rts_v1.cpp
```
#include <iostream>
#include <string>
#include <memory>

class ddtScheme
{
public:
    ddtScheme() {}
};


class EulerDdtScheme
:
    public ddtScheme
{
public:
    EulerDdtScheme() { std::cout << "Construct Euler scheme." << std::endl; }
};


class backwardDdtScheme
:
    public ddtScheme
{
public:
    backwardDdtScheme() { std::cout << "Construct backward scheme." << std::endl; }
};


class ddtSchemeFactory
{
public:
    ddtSchemeFactory() {}

    static std::shared_ptr<ddtScheme> createDdtScheme(const std::string& type)
    {
        if (type == "Euler")
        {
            return std::shared_ptr<ddtScheme>(new EulerDdtScheme);
        }
        else if (type == "backward")
        {
            return std::shared_ptr<ddtScheme>(new backwardDdtScheme);
        }
        else
        {
            std::cout << "Unknown type: " << type << std::endl;
            return nullptr;
        }
    }
};


int main(int argc, char *argv[])
{
    std::shared_ptr<ddtScheme> euler = ddtSchemeFactory::createDdtScheme("Euler");
    std::shared_ptr<ddtScheme> banana = ddtSchemeFactory::createDdtScheme("banana");

    return 0;
}
```
##### mini-rts_v2.cpp
```
#include <iostream>
#include <string>
#include <memory>

class ddtScheme
{
public:
    ddtScheme() {}
};


class EulerDdtScheme
:
    public ddtScheme
{
public:
    EulerDdtScheme() { std::cout << "Construct Euler scheme." << std::endl; }
};


class backwardDdtScheme
:
    public ddtScheme
{
public:
    backwardDdtScheme() { std::cout << "Construct backward scheme." << std::endl; }
};


class steadyStateDdtScheme
:
    public ddtScheme
{
public:
    steadyStateDdtScheme() { std::cout << "Construct steadyState scheme." << std::endl; }
};


class ddtSchemeFactory
{
public:
    ddtSchemeFactory() {}

    static std::shared_ptr<ddtScheme> createDdtScheme(const std::string& type)
    {
        if (type == "Euler")
        {
            return std::shared_ptr<ddtScheme>(new EulerDdtScheme);
        }
        else if (type == "backward")
        {
            return std::shared_ptr<ddtScheme>(new backwardDdtScheme);
        }
        else if (type == "steadyState")
        {
            return std::shared_ptr<ddtScheme>(new steadyStateDdtScheme);
        }
        else
        {
            std::cout << "Unknown type: " << type << std::endl;
            return nullptr;
        }
    }
};


int main(int argc, char *argv[])
{
    std::shared_ptr<ddtScheme> euler = ddtSchemeFactory::createDdtScheme("Euler");
    std::shared_ptr<ddtScheme> steadystate = ddtSchemeFactory::createDdtScheme("steadyState");
    std::shared_ptr<ddtScheme> banana = ddtSchemeFactory::createDdtScheme("banana");

    return 0;
}
```
##### mini-rts_v3.cpp
```
#include <iostream>
#include <string>
#include <memory>
#include <unordered_map>

class ddtScheme
{
public:
    ddtScheme() {}

    // 为构造函数定义类型别名
    typedef std::shared_ptr<ddtScheme> (*constructorPtr)();

    // 为 map 定义类型别名
    typedef std::unordered_map<std::string, constructorPtr> constructorMap;

    // 声明 map 对象指针
    static constructorMap *constructorMapPtr_;

    // 创建 map 对象
    static void constructConstructorMaps()
    {
        static bool constructed = false;
        if (!constructed)
        {
            constructed = true;
            constructorMapPtr_ = new constructorMap;
        }
    }

    template<typename ddtSchemeType>
    static std::shared_ptr<ddtScheme> ddtSchemeNew()
    {
        return std::shared_ptr<ddtScheme>(new ddtSchemeType);
    }

    template<typename ddtSchemeType>
    static void addConstructorToMaps(const std::string& lookup)
    {
        ddtScheme::constructConstructorMaps();
        ddtScheme::constructorMapPtr_->insert(ddtScheme::constructorMap::value_type(lookup, ddtSchemeNew<ddtSchemeType>));
    }
};

ddtScheme::constructorMap *ddtScheme::constructorMapPtr_ = NULL;

class EulerDdtScheme
:
    public ddtScheme
{
public:
    EulerDdtScheme() { std::cout << "Construct Euler scheme." << std::endl; }
};


class backwardDdtScheme
:
    public ddtScheme
{
public:
    backwardDdtScheme() { std::cout << "Construct backward scheme." << std::endl; }
};


class steadyStateDdtScheme
:
    public ddtScheme
{
public:
    steadyStateDdtScheme() { std::cout << "Construct steadyState scheme." << std::endl; }
};


class ddtSchemeFactory
{
public:
    ddtSchemeFactory() {}

    static std::shared_ptr<ddtScheme> createDdtScheme(const std::string& type)
    {
        ddtScheme::constructorMap::iterator it = ddtScheme::constructorMapPtr_->find(type);
        if (it != ddtScheme::constructorMapPtr_->end())
        {
            return it->second();
        }
        else
        {
            std::cout << "Unknown type: " << type << std::endl;
            return nullptr;
        }
    }
};


int main(int argc, char *argv[])
{
    // 初始化 map 对象
    ddtScheme::constructConstructorMaps();

    // 往 map 对象中添加条目
    ddtScheme::addConstructorToMaps<EulerDdtScheme>("Euler");
    ddtScheme::addConstructorToMaps<backwardDdtScheme>("backward");
    ddtScheme::addConstructorToMaps<steadyStateDdtScheme>("steadyState");

    std::shared_ptr<ddtScheme> euler = ddtSchemeFactory::createDdtScheme("Euler");
    std::shared_ptr<ddtScheme> steadystate = ddtSchemeFactory::createDdtScheme("steadyState");
    std::shared_ptr<ddtScheme> banana = ddtSchemeFactory::createDdtScheme("banana");

    return 0;
}
```
###### mini-rts_v4.cpp
```
#include <iostream>
#include <string>
#include <memory>
#include <unordered_map>

class ddtScheme
{
public:
    ddtScheme() {}

    static std::shared_ptr<ddtScheme> New(const std::string& type)
    {
        ddtScheme::constructorMap::iterator it = ddtScheme::constructorMapPtr_->find(type);
        if (it != ddtScheme::constructorMapPtr_->end())
        {
            return it->second();
        }
        else
        {
            std::cout << "Unknown type: " << type << std::endl;
            return nullptr;
        }
    }

    // 为构造函数定义类型别名
    typedef std::shared_ptr<ddtScheme> (*constructorPtr)();

    // 为 map 定义类型别名
    typedef std::unordered_map<std::string, constructorPtr> constructorMap;

    // 声明 map 对象指针
    static constructorMap *constructorMapPtr_;

    // 创建 map 对象
    static void constructConstructorMaps()
    {
        static bool constructed = false;
        if (!constructed)
        {
            constructed = true;
            constructorMapPtr_ = new constructorMap;
        }
    }

    // 用于实现初始化 map 对象的类模板，定义该类的静态变量以实现自动添加条目至 map 对象
    template<typename ddtSchemeType>
    class addConstructorToMap
    {
    public:
        static std::shared_ptr<ddtScheme> ddtSchemeNew()
        {
            return std::shared_ptr<ddtScheme>(new ddtSchemeType);
        }

        addConstructorToMap(const std::string& lookup = ddtSchemeType::typeName)
        {
            ddtScheme::constructConstructorMaps();
            ddtScheme::constructorMapPtr_->insert(ddtScheme::constructorMap::value_type(lookup, ddtSchemeNew));
        }
    };
};

ddtScheme::constructorMap *ddtScheme::constructorMapPtr_ = NULL;

class EulerDdtScheme
:
    public ddtScheme
{
public:
    static const std::string typeName;
    EulerDdtScheme() { std::cout << "Construct Euler scheme." << std::endl; }
};


class backwardDdtScheme
:
    public ddtScheme
{
public:
    static const std::string typeName;
    backwardDdtScheme() { std::cout << "Construct backward scheme." << std::endl; }
};


class steadyStateDdtScheme
:
    public ddtScheme
{
public:
    static const std::string typeName;
    steadyStateDdtScheme() { std::cout << "Construct steadyState scheme." << std::endl; }
};

// 定义 typeName
const std::string EulerDdtScheme::typeName = "Euler";
const std::string backwardDdtScheme::typeName = "backward";
const std::string steadyStateDdtScheme::typeName = "steadyState";

// 用模板特化后的类定义静态变量
ddtScheme::addConstructorToMap<EulerDdtScheme> addEulerConstructorToTable_;
ddtScheme::addConstructorToMap<backwardDdtScheme> addbackwardConstructorToTable_;
ddtScheme::addConstructorToMap<steadyStateDdtScheme> addsteadyStateConstructorToTable_;


int main(int argc, char *argv[])
{
    std::shared_ptr<ddtScheme> euler = ddtScheme::New("Euler");
    std::shared_ptr<ddtScheme> steadystate = ddtScheme::New("steadyState");
    std::shared_ptr<ddtScheme> banana = ddtScheme::New("banana");

    return 0;
}
```
原文链接：<https://marinecfd.xyz/post/mini-runtime-selection-with-cpp11/>
 


 <span id="busuanzi_container_page_pv">
  本文总阅读量<span id="busuanzi_value_page_pv"></span>次
</span>

