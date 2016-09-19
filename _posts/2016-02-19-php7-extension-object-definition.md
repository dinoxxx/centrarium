---
layout: post
title:  "PHP7扩展开发 - 对象定义"
date:   2016-02-19
author: 刘宇
categories: PHP
tags:	php
---

上一篇介绍了扩展里面方法的定义，我们都知道，PHP不仅支持面向过程，还支持面向对象，这一篇继续介绍对象定义。

## 一、最简单的类

我们先定义一个最简单的类，简单到什么程度呢，只能new一个对象，没有属性，没有方法，先把路走通。我们先按照第一篇的内容生成一个people扩展，现在准备定义一个People类。

```
zend_class_entry *people_ce;

const zend_function_entry people_functions[] = {
    PHP_FE_END  /* Must be the last line in people_functions[] */
};

PHP_MINIT_FUNCTION(people)
{
    zend_class_entry ce;
    INIT_CLASS_ENTRY(ce, "People", people_functions);
    people_ce = zend_register_internal_class(&ce TSRMLS_CC);
    return SUCCESS;
}
```

然后在PHP代码里面测试一下，可以看到

```
<?php
$people = new People();
var_dump($people);
/* Output
object(People)#1 (0) {
}
*/
```

`zend_class_entry`是Zend API里面关于类的定义的核心，想定义一个类就先初始化这么一个结构体变量，在Zend/zend.h中可以看到这个结构体的定义，这里先捡一些重要的介绍。`INIT_CLASS_ENTRY`帮助我们初始化一个类的定义，第一个参数是`zend_class_entry`变量，第二个参数是类的名字，第三个参数是类的方法集合。然后是`zend_register_internal_class`方法帮助我们将这个类的定义注册一下。一般情况下，`zend_class_entry`变量在别的地方会用到，所以定义一个全局变量放在外面，`zend_register_internal_class`会返回结构体指针赋值给people_ce。以上这些工作都需要放在扩展的模块初始化方法`PHP_MINIT_FUNCTION`里面。

```
struct _zend_class_entry {
    /* 类的类型：ZEND_INTERNAL_CLASS 或者 ZEND_USER_CLASS */
    char type;
    /* 类的名称 */
    zend_string *name;
    /* 父类 */
    struct _zend_class_entry *parent;
    /* 引用计数 */
    int refcount;
    /*
    * ce_flags指定了类的基本属性
    * ZEND_ACC_IMPLICIT_ABSTRACT_CLASS 隐式的抽象类（含有abstract方法）
    * ZEND_ACC_EXPLICIT_ABSTRACT_CLASS 添加了abstract关键字的抽象类
    * ZEND_ACC_FINAL_CLASS 该类是final的
    * ZEND_ACC_INTERFACE 这是一个接口
    */
    uint32_t ce_flags;

    /* 下面5个属性暂时不知道含义，待补充 */
    int default_properties_count;
    int default_static_members_count;
    zval *default_properties_table;
    zval *default_static_members_table;
    zval *static_members_table;
    /* 方法哈希表 */
    HashTable function_table;
    /* 属性信息哈希表 */
    HashTable properties_info;
    /* 常量哈希表 */
    HashTable constants_table;

    /* 构造函数、析构函数、clone函数以及其它魔术方法 */
    union _zend_function *constructor;
    union _zend_function *destructor;
    union _zend_function *clone;
    union _zend_function *__get;
    union _zend_function *__set;
    union _zend_function *__unset;
    union _zend_function *__isset;
    union _zend_function *__call;
    union _zend_function *__callstatic;
    union _zend_function *__tostring;
    union _zend_function *__debugInfo;
    ......
};
```

## 二、添加属性

上面的类只有一个空壳，我们常见的类都是有属性和方法的。添加属性有以下这些方法，从名字就可以看出来添加不同类型的属性用不同的方法。

```
ZEND_API int zend_declare_property_ex(zend_class_entry *ce, zend_string *name, zval *property, int access_type, zend_string *doc_comment);
ZEND_API int zend_declare_property(zend_class_entry *ce, const char *name, size_t name_length, zval *property, int access_type);
ZEND_API int zend_declare_property_null(zend_class_entry *ce, const char *name, size_t name_length, int access_type);
ZEND_API int zend_declare_property_bool(zend_class_entry *ce, const char *name, size_t name_length, zend_long value, int access_type);
ZEND_API int zend_declare_property_long(zend_class_entry *ce, const char *name, size_t name_length, zend_long value, int access_type);
ZEND_API int zend_declare_property_double(zend_class_entry *ce, const char *name, size_t name_length, double value, int access_type);
ZEND_API int zend_declare_property_string(zend_class_entry *ce, const char *name, size_t name_length, const char *value, int access_type);
ZEND_API int zend_declare_property_stringl(zend_class_entry *ce, const char *name, size_t name_length, const char *value, size_t value_len, int access_type);
```

如果想添加一个字符串类型的public的非静态属性"name"，复用上面的部分示例代码，使用`zend_declare_property_string`这个方法，第一个参数是类的结构体变量，第二个参数是属性名称，第三个参数是属性长度，第四个参数是默认值，第五个参数是访问权限。其他添加属性的方法都是类似的用法。

```
PHP_MINIT_FUNCTION(people)
{
    zend_class_entry ce;
    INIT_CLASS_ENTRY(ce, "People", people_functions);
    people_ce = zend_register_internal_class(&ce TSRMLS_CC);
    zend_declare_property_string(people_ce, "name", strlen("name"), "", ZEND_ACC_PUBLIC TSRMLS_CC);

    return SUCCESS;
}
```

如果想添加一个public的静态属性，其他都是一样的，只需要修改一下访问权限这个参数。

```
zend_declare_property_string(people_ce, "name", strlen("name"), "", ZEND_ACC_PUBLIC|ZEND_ACC_STATIC TSRMLS_CC);
```

全部访问权限如下所示，注释中标明了使用的地方，`fn_flags`代表可以在定义方法时使用，`zend_property_info.flags`代表可以在定义属性时使用，`ce_flags`代表在定义类时使用。

```
#define ZEND_ACC_STATIC                     0x01     /* fn_flags, zend_property_info.flags */
#define ZEND_ACC_ABSTRACT                   0x02     /* fn_flags */
#define ZEND_ACC_FINAL                      0x04     /* fn_flags */
#define ZEND_ACC_IMPLEMENTED_ABSTRACT       0x08     /* fn_flags */
#define ZEND_ACC_IMPLICIT_ABSTRACT_CLASS    0x10     /* ce_flags */
#define ZEND_ACC_EXPLICIT_ABSTRACT_CLASS    0x20     /* ce_flags */
#define ZEND_ACC_FINAL_CLASS                0x40     /* ce_flags */
#define ZEND_ACC_INTERFACE                  0x80     /* ce_flags */
#define ZEND_ACC_INTERACTIVE                0x10     /* fn_flags */
#define ZEND_ACC_PUBLIC                     0x100    /* fn_flags, zend_property_info.flags */
#define ZEND_ACC_PROTECTED                  0x200    /* fn_flags, zend_property_info.flags */
#define ZEND_ACC_PRIVATE                    0x400    /* fn_flags, zend_property_info.flags */
#define ZEND_ACC_PPP_MASK   (ZEND_ACC_PUBLIC | ZEND_ACC_PROTECTED | ZEND_ACC_PRIVATE)
#define ZEND_ACC_CHANGED                    0x800    /* fn_flags, zend_property_info.flags */
#define ZEND_ACC_IMPLICIT_PUBLIC            0x1000   /* zend_property_info.flags; unused (1) */
#define ZEND_ACC_CTOR                       0x2000   /* fn_flags */
#define ZEND_ACC_DTOR                       0x4000   /* fn_flags */
#define ZEND_ACC_CLONE                      0x8000   /* fn_flags */
#define ZEND_ACC_ALLOW_STATIC               0x10000  /* fn_flags */
#define ZEND_ACC_SHADOW                     0x20000  /* fn_flags */
#define ZEND_ACC_DEPRECATED                 0x40000  /* fn_flags */
#define ZEND_ACC_CLOSURE                    0x100000 /* fn_flags */
#define ZEND_ACC_CALL_VIA_HANDLER           0x200000 /* fn_flags */
```

## 三、添加方法

在自定义的类里面添加方法，和上一篇中介绍的自定义的普通方法类似，在上面我们已经看到了初始化的宏的第三个参数是一个方法集合，现在只需要把类的方法定义好，然后放在这个集合里面即可，下面看代码。

```
ZEND_METHOD(People, __construct)
{
    php_printf("construct method\n");
}
ZEND_METHOD(People, __destruct)
{
    php_printf("destruct method\n");
}
ZEND_METHOD(People, talk)
{
    php_printf("public method talk\n");
}

const zend_function_entry people_functions[] = {
    ZEND_ME(People, __construct, NULL, ZEND_ACC_PUBLIC|ZEND_ACC_CTOR)
    ZEND_ME(People, __destruct, NULL, ZEND_ACC_PUBLIC|ZEND_ACC_DTOR)
    ZEND_ME(People, talk, NULL, ZEND_ACC_PUBLIC|ZEND_ACC_STATIC)
    PHP_FE_END  /* Must be the last line in people_functions[] */
};
```

可以很明显地看出，和上一篇中使用的宏不同了，`PHP_FUNCTION`变成了`ZEND_METHOD`，`PHP_FE`变成了`ZEND_ME`，而且它们的参数有很大的不同，我们先想当然地认为后者是为了给类添加方法用的，后面有机会再分析这些宏的区别。上面的示例代码，我特意写了构造函数和析构函数这种魔术方法，还写了talk这个普通方法，在方法集合中添加的时候，访问权限就有了明显的不同，这里`ZEND_ACC_CTOR`特指构造方法，`ZEND_ACC_DTOR`特指析构方法。上面列出的全部权限中，注释里面标有`fn_flags`的都可以在这里使用。

## 三、添加常量

如果想在类里面自定义常量，Zend API同样给了一组方法，在Zend/zend_API.h中可以找到。参数和上面定义类的属性基本一致，不再赘述。

```
ZEND_API int zend_declare_class_constant(zend_class_entry *ce, const char *name, size_t name_length, zval *value);
ZEND_API int zend_declare_class_constant_null(zend_class_entry *ce, const char *name, size_t name_length);
ZEND_API int zend_declare_class_constant_long(zend_class_entry *ce, const char *name, size_t name_length, zend_long value);
ZEND_API int zend_declare_class_constant_bool(zend_class_entry *ce, const char *name, size_t name_length, zend_bool value);
ZEND_API int zend_declare_class_constant_double(zend_class_entry *ce, const char *name, size_t name_length, double value);
ZEND_API int zend_declare_class_constant_stringl(zend_class_entry *ce, const char *name, size_t name_length, const char *value, size_t value_length);
ZEND_API int zend_declare_class_constant_string(zend_class_entry *ce, const char *name, size_t name_length, const char *value);
```

再举一个示例，方便大家理解，给我们自定义的People类添加一个常量"AGE"，赋值18，在PHP代码中就可以直接通过People::AGE来使用这个常量。

```
zend_declare_class_constant_long(people_ce, "AGE", strlen("AGE"), 18 TSRMLS_DC);
```
