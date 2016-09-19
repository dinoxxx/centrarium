---
layout: post
title:  "如何把扩展从PHP5升级到PHP7"
date:   2016-05-25
author: 刘宇
categories: PHP
tags:	php
---

我在公司的生产环境已经升级了PHP7，大部分活跃的扩展都可以兼容PHP7或者有了PHP7的分支，但是处理protocolbuffers数据的扩展一直没有人来升级，我只能摸着石头过河，自己做一把升级了。目前已经编译通过，处在处理单测失败的case阶段，欢迎大家一起讨论。对PHP扩展升级可以先看一下官方的升级说明，https://wiki.php.net/phpng-upgrading，基本可以应对大部分场景，剩下的就需要自己读源代码了。

## zval

zval结构体是Zend内核的非常核心的结构，在PHP5和PHP7之间的差别非常大，我给出2处文章供大家学习，基本上可以代表这块知识点最权威的介绍了。

*深入理解PHP7之zval（鸟哥）*
https://github.com/laruence/php7-internal/blob/master/zval.md

*变量在 PHP7 内部的实现（Nikita Popov）中文版*
http://0x1.im/blog/php/Internal-value-representation-in-PHP-7-part-1.html
http://0x1.im/blog/php/Internal-value-representation-in-PHP-7-part-2.html

PHP7不再使用zval的二级指针，大多数场景下出现的zval**变量都改成zval*，相应的使用在这些变量上的宏Z_*_PP也需要改成Z_*_P。
在大部分场景下，PHP7是在栈上直接使用zval，不需要去堆上分配内存。这时，zval *就需要改成zval，宏也需要从Z_*_P改成Z_*，创建宏从ZVAL_*(var)转换成ZVAL_*(&var)。所以，分配zval内存的宏

```
ALLOC_ZVAL、ALLOC_INIT_ZVAL、MAKE_STD_ZVAL都被删掉了。
-  zval *zv;
-  MAKE_STD_ZVAL(zv);
-  array_init(zv);
+  zval zv;
+  array_init(&zv);
```

PHP7中zval的long和double类型是不需要引用计数的，所以相关的宏要做调整。

```
- Z_ADDREF_P(zv)
+ Z_TRY_ADDREF_P(zv);
```

PHP7中zval的类型，删除了IS_BOOL，增加了IS_TRUE和IS_FALSE。

```
- if (Z_TYPE_P(zv) == IS_BOOL) {
- }
+ if (Z_TYPE_P(zv) == IS_TRUE) {
+ } else if (Z_TYPE_P(zv) == IS_FALSE) {
+ }
```

## zend_string

PHP7中增加了一个新的内置字符串类型zend_string，下面是Zend内核中的结构体定义。

```
struct _zend_string {
    zend_refcounted_h gc;     /* 垃圾回收结构体 */
    zend_ulong        h;      /* 字符串哈希值 */
    size_t            len;    /* 字符串长度 */
    char              val[1]; /* 字符串内容 */
};
```

gc是PHP7中的所有非标量结构都包含的垃圾回收结构体变量；h是字符串哈希值，作为HashTable的key时不需要每次都重新计算哈希值，提高了效率；len是字符串长度，同理每次使用到字符串的长度时不需要再计算，提高了效率；val[1]是C语言的黑科技，此处按照char *理解即可。这里有三个宏帮助我们方便的使用zend_string的变量。

```
#define ZSTR_VAL(zstr)  (zstr)->val
#define ZSTR_LEN(zstr)  (zstr)->len
#define ZSTR_H(zstr)    (zstr)->h
```

创建和销毁zend_string使用以下方法。

```
zend_string *zend_string_init(const char *str, size_t len, int persistent)
void zend_string_release(zend_string *s)
```

zend_string用来替代PHP5中使用char *和int的场景，尤其是很多API的参数和返回值都做了调整。

```
- int zend_hash_find(const HashTable *ht, const char *arKey, uint nKeyLength, void **pData)
+ zval* ZEND_FASTCALL zend_hash_find(const HashTable *ht, zend_string *key)

- void zend_mangle_property_name(char **dest, int *dest_length, const char *src1, int src1_length, const char *src2, int src2_length, int internal);
+ zend_string *zend_mangle_property_name(const char *src1, size_t src1_length, const char *src2, size_t src2_length, int internal)
```

## HashTable API

在PHP7中使用HashTable的API方法时，有了非常明显的变化。
查询方法，PHP5使用引用传参的方式，同时返回SUCCESS/FAILURE；PHP7直接返回结果，查询无结果时返回NULL。

```
- int zend_hash_find(const HashTable *ht, const char *arKey, uint nKeyLength, void **pData)
+ zval* ZEND_FASTCALL zend_hash_find(const HashTable *ht, zend_string *key)
```

HashTable的API方法中的key，PHP5中使用char *和int代表的字符串；PHP7中使用zend_string代表的字符串，同时提供了对char *和int支持的一组方法，但是需要注意的是这里的字符串长度是不包括结尾的'\0'的，在升级扩展时难免会碰到很多地方需要加减一。

```
- int zend_hash_exists(const HashTable *ht, const char *arKey, uint nKeyLength)
+ zend_bool zend_hash_exists(const HashTable *ht, zend_string *key)
+ zend_bool zend_hash_str_exists(const HashTable *ht, const char *str, size_t len)
```

PHP7为HashTable的value为指针时设计了一组API，在常规的API方法后添加后缀_ptr即可。

```
void *zend_hash_find_ptr(const HashTable *ht, zend_string *key)
void *zend_hash_update_ptr(HashTable *ht, zend_string *key, void *pData)
```

PHP7为HashTable的轮询设计了一组宏，使用起来非常方便。

```
ZEND_HASH_FOREACH_VAL(ht, val)
ZEND_HASH_FOREACH_KEY(ht, h, key)
ZEND_HASH_FOREACH_PTR(ht, ptr)
ZEND_HASH_FOREACH_NUM_KEY(ht, h)
ZEND_HASH_FOREACH_STR_KEY(ht, key)
ZEND_HASH_FOREACH_STR_KEY_VAL(ht, key, val)
ZEND_HASH_FOREACH_KEY_VAL(ht, h, key, val)
```

## 自定义对象

这里有点复杂，我直接附上我的代码，结合代码来做详细说明。

```
typedef struct{
    int max;
    int offset;
    zend_object zo;
} php_protocolbuffers_unknown_field_set;

static zend_object_handlers php_protocolbuffers_unknown_field_set_object_handlers;

static void php_protocolbuffers_unknown_field_set_free_storage(php_protocolbuffers_unknown_field_set *object TSRMLS_DC)
{
    php_protocolbuffers_unknown_field_set *unknown_field_set;
    unknown_field_set = (php_protocolbuffers_unknown_field_set*)((char *) object - XtOffsetOf(php_protocolbuffers_unknown_field_set, zo))；

    zend_object_std_dtor(&unknown_field_set->zo TSRMLS_CC);
}

zend_object *php_protocolbuffers_unknown_field_set_new(zend_class_entry *ce TSRMLS_DC)
{
    php_protocolbuffers_unknown_field_set *intern;
    intern = ecalloc(1, sizeof(php_protocolbuffers_unknown_field_set) + zend_object_properties_size(ce));
    zend_object_std_init(&intern->zo, ce);
    object_properties_init(&intern->zo, ce);
    intern->zo.handlers = &php_protocolbuffers_unknown_field_set_object_handlers;
    
    intern->max    = 0; 
    intern->offset = 0;
    
    return &intern->zo;
}

void php_protocolbuffers_unknown_field_set_class(TSRMLS_D)
{
// 此处有省略
    php_protocol_buffers_unknown_field_set_class_entry->create_object = php_protocolbuffers_unknown_field_set_new;
    memcpy(&php_protocolbuffers_unknown_field_set_object_handlers, zend_get_std_object_handlers(), sizeof(zend_object_handlers));
    php_protocolbuffers_unknown_field_set_object_handlers.offset = XtOffsetOf(php_protocolbuffers_unknown_field_set, zo);
    php_protocolbuffers_unknown_field_set_object_handlers.free_obj = php_protocolbuffers_unknown_field_set_free_storage;
}
```

我们想自定义一个php_protocolbuffers_unknown_field_set的对象，在它的结构体里面除了zend_object，还有自定义的max和offset，务必把zend_object放在最后。
实际生成对象的地方基本就是标准写法，先分配内存，包括php_protocolbuffers_unknown_field_set结构体的内存和对象属性的内存；然后对zend_object的handlers赋值；最后再对自己自定义的变量初始化。
实际生成对象handler的地方也是标准写法，先分配内存，offset是必须设置的，可选的设置项有free_obj，dtor_obj，clone_obj。
想取到zend_object，需要(STRUCT_NAME *)((char *)OBJECT - XtOffsetOf(STRUCT_NAME, zo))

















