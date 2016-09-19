---
layout: post
title:  "如何在PHP中处理Protocol Buffers数据"
date:   2016-05-05
author: 刘宇
categories: PHP
tags:	php
---

我们知道Protocol Buffers是Google定义的一种跨语言、跨平台、可扩展的数据传输及存储的协议，因为将字段协议分别放在传输两端，传输数据中只包含数据本身，不需要包含字段说明，所以传输数据量小，解析效率高。感兴趣的可以访问https://developers.google.com/protocol-buffers/docs/overview。Protocol Buffers官方只支持C++, Java, Python, C#, Go，如果想在PHP中使用Protocol Buffers，需要借助于第三方的扩展，使用方法如下。<!-- more -->

## 安装protoc编译器

第一步，安装Google的protoc编译器，这个工具可以把proto文件中定义的Message转换为各种编程语言中的类。下载release版本直接编译安装。

```
tar -xvzf protobuf-2.5.0.tar.gz
cd protobuf-2.5.0
./configure --prefix=/usr/local/protobuf
make && make install
```

[下载地址](https://github.com/google/protobuf/releases) 
 
## 安装PHP Plugin

第二步，安装protoc的PHP plugin，需要使用composer安装，感兴趣的可以访问http://www.phpcomposer.com
这个插件可以将proto文件转换到PHP文件，在PHP应用中引用后，可以将二进制格式的Protocol Buffers数据转换为PHP的对象

```
unzip protoc-gen-php-master.zip
cd protoc-gen-php-master
curl -sS https://getcomposer.org/installer | php
php composer.phar install
/usr/local/protobuf/bin/protoc --plugin=bin/protoc-gen-php --php_out=php_kingso -I. kingso.proto
```

[下载地址](https://github.com/chobie/protoc-gen-php)
 
## 安装php-protocolbuffers

第三步，安装php-protocolbuffers，这是一个PHP扩展，在第二步中已经完成了proto文件到PHP文件的转换，但是对PHP对象的各种操作还需要这个扩展中的API方法

```
phpize
./configure
make && make install
```

然后在你的php.ini配置文件中添加

```
extension = "protocolbuffers.so"
```

[下载地址](https://github.com/chobie/php-protocolbuffers)

## 编写代码

第四步，借助上面提到的扩展的各种API方法，可以简单地get出自己需要的字段完成后续工作

```
require "....../kingso.proto.php"
$ks_result = KSResult::parseFromString($kingso_res);
$ks_result->get('xxx_name');
```
