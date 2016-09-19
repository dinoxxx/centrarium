---
layout: post
title:  "PHP7扩展开发 - 方法定义"
date:   2016-02-18
author: 刘宇
categories: PHP
tags:	php
---

前面介绍了扩展的基本组成，并且实现了"Hello World"，为了更进一步接近我们的实际需要，这一篇介绍一下扩展的方法定义。

## 一、方法的输入

第一个hello_world方法只是简单的打印字符串，并没有输入参数，现在我们定义一个hello_world_with_input方法

```
PHP_FUNCTION(hello_world_with_input)
{
    char *arg1 = NULL, *arg2 = NULL;
    size_t arg_len1, arg_len2;
    if (zend_parse_parameters(ZEND_NUM_ARGS(), "ss", &arg1, &arg_len1, &arg2, &arg_len2) == FAILURE) {
        return;
    }
    php_printf("Hello World, %s and %s!\n", arg1, arg2);
    RETURN_TRUE;
}
```

这个方法有两个输入参数，需要使用zend_parse_parameters方法读出参数，方法的原型是

```
int zend_parse_parameters(int num_args, const char *type_spec, ...);
```

该方法的返回值，FAILURE 表示为 -1，SUCCESS 表示为 0。
第一个参数表示自定义的扩展方法的参数个数，一般使用ZEND_NUM_ARGS()来表示
第二个参数表示参数的格式，常用的说明在下面，全部相关说明可以阅读PHP7源代码压缩包下面的README.PARAMETER_PARSING_API
后面数量可变的参数就是方法的读入变量指针了，此处传入的参数是字符串，需要2个变量指针，分别读入字符串的值和字符串的长度。如果是整型数或者浮点数，用一个变量指针就可以了。

| 修饰符 | 附加参数的类型 | 描述 |
|:------:|:--------------:|:----:|
| l | long | int, long值 |
| d | double | float, double值 |
| s | char*, int | 字符串 |
| b | boolean | 布尔值 |

## 二、方法的返回

我们再定义一个hello_world_with_output方法，来介绍一下扩展方法如何输出。

```
PHP_FUNCTION(hello_world_with_output)
{
    char *arg = NULL;
    size_t arg_len;
    zend_string *strg;
    if (zend_parse_parameters(ZEND_NUM_ARGS(), "s", &arg, &arg_len) == FAILURE) {
        return;
    }
    strg = strpprintf(0 ,"Hello World, %s!", arg);
    RETURN_STR(strg);
}
```

一般情况下，返回用各种数据类型的宏来完成，此处返回的是一个字符串，用到的宏是RETURN_STR()。扩展API包含丰富的用于从函数中返回值的宏。这些宏有两种主要风格：第一种是RETVAL_type()形式，它设置了返回值但C代码继续执行。这通常使用在把控制交给脚本引擎前还希望做的一些清理工作的时候使用，然后再使用C的返回声明 "return"返回到PHP；后一个宏更加普遍，其形式是RETURN_type()，他设置了返回类型，同时返回控制到PHP。下表解释了常用的宏，全部相关说明可以阅读Zend目录下的zend_API.h。


| 设置返回值并且结束函数 | 设置返回值 | 宏返回类型和参数 |
|:----------------------:|:----------:|:----------------:|
| RETURN_LONG(l) | RETVAL_LONG(l) | 整数 |
| RETURN_DOUBLE(d) | RETVAL_DOUBLE(d) | 浮点数 |
| RETURN_STR(s) | RETVAL_STR(s) | 字符串 |
| RETURN_BOOL(b) | RETVAL_BOOL(b) | 布尔值 |

注意，上面介绍的两个方法，定义之后想测试效果，还需要在hello_functions数组中添加一下。

## 三、头文件中添加方法声明

C语言中默认的规则是把方法的声明放在头文件中，所以我们在php_hello.h中添加

```
PHP_FUNCTION(hello_world_with_input);
PHP_FUNCTION(hello_world_with_output);
```
