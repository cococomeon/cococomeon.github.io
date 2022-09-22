---
layout: post
title: gradle dsl
subtitle:
categories: Gradle
tags: [dsl]
banner: "/assets/images/banners/home.jpeg"

---

## **一. 简介**

gradle脚本是是以groovy脚本语言为基础， 基于DSL语法的自动化构建工具

*DSL(domain-specific language) 领域特定语言， 指的是针对某领域的特定编程语言， 通常没有java，c， c++这么复杂，而是将摸个领域的对象作为编程的元素，所以使用领域特定于语言的一般也是特定领域的人，然后再由对应的DSL程序翻译成源代码。实际就是对应领域的一个编译器编译领域语言成源码的过程*

*gradle是基于groovy实现的，gradle的构建脚本就是以groovy构建的适用于gradle的DSL语言。gradle的DSL语言主要有project和task两个概念，然后还有通过插件plugin实现了gradle dsl语言的*



## **二. 预备知识**



1. Groovy开发DLS的支持和方便

```groovy
show = { println it }            //1. groovy 闭包， 匿名函数 语法{}
square_root = { Math.sqrt(it) }	 //2. it为groovy保留关键字， 类似java的this，代表的是闭包的入参

def please(action) {             //3. 定义一个函数， 省略了return关键字， []是map语法，the是字符串key，但是没有使用引号
  [the: { what ->                //4. 闭包中的函数也省略了return关键字，函数与闭包中不写return语句时默认是将最后面的那个语句作为返回值return
    [of: { n -> action(what(n)) }]
  }]
}

//上面的函数等价于下面的
def please(action){
  return [
    "the": {
      "what" -> return [
        "of": {
          n -> return action(what(n))
        }
      ]
    }
  ];
}
// 等同于：please(show).the(square_root).of(100)
// please(show)
please show the square_root of 100   //groovy支持在同一行中方法调用可以省略括号和点号
// ==> 10.0
```

2. gradle中使用到的groovy特性
   1. 闭包 {}， 作为	函数的调用入参
   2. 接口中getXxxx方法可以直接直接使用xxxx以属性作为调用



gradle实战

```groovy
plugins {
	id "org.springframework.boot.starter"
}

description = "Starter for using JDBC with the HikariCP connection pool"

dependencies { //调用的是gradle-api包中的 Project接口的 void dependencies(Closure var1) 接口， 语法就是省略了(), 通过groovy语法， dependencies {}的方式调用了 void dependencies(Closure var1)这个方法
	api(project(":spring-boot-project:spring-boot-starters:spring-boot-starter"))
	api("com.zaxxer:HikariCP")
	api("org.springframework:spring-jdbc")
}
```

gradle的实现是使用java代码写的，groovy可以通过自己的语法进行调用gradle中的实现，build.gradle脚本就是一个groovy语法的脚本，对gradle的调用。




