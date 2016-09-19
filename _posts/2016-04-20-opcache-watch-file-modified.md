---
layout: post
title:  "PHP造日志轮子的经验"
date:   2016-04-20
author: 刘宇
categories: PHP
tags:	php
---

昨晚线上出故障，紧急处理切换容灾后缓解了故障，解决故障后从容灾切换回正式服务时发现PHP文件更新无效，重启FPM后才生效。下面记录复盘追查的过程。

因为是PHP文件更新不生效，所以马上怀疑到opcache上面，到线上看了一眼php.ini，果然使用了opcache，并且检测间隔时间设置为60秒。查看昨晚的日志，更新不生效持续时间远远大于60秒，所以这个检测间隔时间的问题可以PASS了，我们继续。

在线上环境查看更新的文件和日志中的时间的时候，突然发现PHP文件时间和日志中的时间对应不上，马上找OP确认，OP交待说这个文件是回滚mv回来的，所以文件时间和我预期的不一致。我用stat命令看了一下，果然modify time相比access time早了一段时间，依据这点线索推测opcache依靠的是PHP文件的modify time作为文件被修改的检测条件。在线下复现问题，填坑成功！

下面总结一下填坑过程中查的一些相关的知识点

### 加载opcache

在php.ini中增加opcache时需要使用zend_extension，而不是extension，不然会得到以下WARNING

```
PHP Warning:  PHP Startup: Invalid library (appears to be a Zend Extension, try loading using zend_extension=opcache.so from php.ini) in Unknown on line 0
```

### 配置opcache

使用下列推荐设置来获得较好的性能： 

```
opcache.memory_consumption=128
opcache.interned_strings_buffer=8
opcache.max_accelerated_files=4000
opcache.revalidate_freq=60
opcache.fast_shutdown=1
opcache.enable_cli=1
```

本次涉及到的有两个参数

revalidate_freq，默认2
检查脚本时间戳是否有更新的周期，以秒为单位。 设置为 0 会导致针对每个请求， opcache 都会检查脚本更新

validate_timestamps，默认1
如果启用，那么 OPcache 会每隔 opcache.revalidate_freq 设定的秒数 检查脚本是否更新。 如果禁用此选项，你必须使用 opcache_reset() 或者 opcache_invalidate() 函数来手动重置 opcache，也可以 通过重启 Web 服务器来使文件系统更改生效。

### 系统命令和函数stat

access time 表示我们最后一次访问文件的时间
modify time 表示我们最后一次修改文件的时间
change time 表示我们最后一次对文件属性改变的时间，包括权限，大小，属性等等

C/C++中也可以使用stat方法查询文件的这3个时间属性，一般应用都会通过modify time判断文件是否更新，我们本次踩坑就是因为文件是mv操作的，modity time没有更新，所以opcache没有更新。
