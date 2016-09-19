---
layout: post
title:  "用C++开发PHP扩展"
date:   2016-04-27
author: 刘宇
categories: PHP
tags:	php
---

由于PHP的底层就是C开发的，不可避免的常用PHP扩展也都是C开发的，但是有时我们想用C++来开发可以吗，答案当然是可以的，并且有它自己的优势，第一可以方便地面向对象编程，第二可以利用现有C++编译的动态链接库。

常规的PHP扩展开发流程我再别的文章里面已经介绍过了，这里不再赘述，只介绍一下C++开发PHP扩展的不同之处。

## 修改config.m4

利用ext_skel工具生成扩展的基础框架，默认生成的框架是针对C的，所以针对C++修改config.m4文件
config.m4文件是编译基础中最核心的文件，这个文件主要是用autoconf来产生configure配置文件，继而自动生成大家所熟悉的Makefile文件。需要注意的是，每次修改config.m4，需要phpize --clean，再重新phpize

```
PHP_ARG_WITH(dict, for hsdt support,
Make sure that the comment is aligned:
[  --with-demo             Include demo support])
```

表示demo扩展需要依赖外部动态链接库，在configure的时候 --with-demo的参数表示依赖外部动态链接库的路径，比如编译PHP时使用的--with-curl=/usr/local/libcurl表示依赖的libcurl.so的路径在/usr/local/libcurl里面

```
PHP_ADD_INCLUDE($DEMO_DIR/include)
```

表示依赖的外部动态链接库的include的头文件的路径

```
PHP_REQUIRE_CXX()
```

表示这个扩展使用C++

```
PHP_SUBST(DEMO_SHARED_LIBADD)
```

用于说明这个扩展编译成动态链接库的形式

```
PHP_ADD_LIBRARY(stdc++, 1, DEMO_SHARED_LIBADD)
```

用于将标准C++库加入扩展

```
PHP_ADD_LIBRARY_WITH_PATH($LIBNAME, $DICT_DIR/lib64, DICT_SHARED_LIBADD)
```

用于将依赖的外部动态链接库加入扩展

```
PHP_NEW_EXTENSION(demo, 
    xxx.cpp yyy.cpp zzz.cpp,
    $ext_shared,, 
    -DZEND_ENABLE_STATIC_TSRMLS_CACHE=1)
```

第2行指定哪些源文件需要编译，中间用空格间隔

## 修改源文件

包括.h文件和.cpp文件，因为PHP提供的ZEND API都是C编写的，所以在include的时候需要在外面加一层extern "C"，目的是把一些C写的库或宏用兼容的方式给解决。剩下的代码自己用C++自由发挥吧。

```
extern "C" {
  #ifdef ZTS
  #include "TSRM.h"
  #endif
}
```

```
extern "C" {
  #ifdef HAVE_CONFIG_H
  #include "config.h"
  #endif
  #include "php.h"
  #include "php_ini.h"
  #include "ext/standard/info.h"
}
```
