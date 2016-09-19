---
layout: post
title:  "PHP造日志轮子的经验"
date:   2016-03-24
author: 刘宇
categories: PHP
tags:	php
---

最近准备升级PHP7，发现同时使用yaf和seaslog扩展时会导致流量上升时php-fpm子进程的crash，在php-fpm.log中可以看到以下warning记录，最终引起请求中断。

```
WARNING: [pool www] child 15148 exited on signal 6 (SIGABRT) after 337.885989 seconds from start
```

经过动手实验，发现只要加载了seaslog.so，即使不调用它的方法，仍然存在上述问题，推测是seaslog扩展的RINIT和RSHUTDOWN里面的处理有问题，但是检查不出来是什么问题。对于PHP的扩展开发是心有余而力不足，所以只能放弃心爱的seaslog，自己来造个简单的日志轮子了。

以上是故事背景，下面开始讲造轮子的收获。

第一步，简单地实现功能，对文件进行写操作。

```
$fp = fopen($file, 'a');
fwrite($fp, $log);
fclose($fp);
```

第二步，考虑文件锁，高并发场景下有可能会把日志写乱。

```
$fp = fopen($file, 'a');
if (flock($fp, LOCK_EX)) {
    fwrite($fp, $log);
    flock($fp, LOCK_UN);
}
fclose($fp);
```

第三步，考虑到写日志只是一个很简单的应用场景，不需要考虑读文件时的数据一致性，为了提高效率我们可以改良一下这个文件锁。假设程序所在的文件系统的块的空间大小是4096字节，小于这个这个长度的日志可以直接写文件，否则要先抢占文件锁再写文件。

```
$fp = fopen($file, 'a');
if (strlen($log) <= 4096) {
    fwrite($fp, $log);
} else if (flock($fp, LOCK_EX)) {
    fwrite($fp, $log);
    flock($fp, LOCK_UN);
}
fclose($fp);
```

>**If handle was fopen()ed in append mode, fwrite()s are atomic (unless the size of string exceeds the filesystem's block size, on some platforms, and as long as the file is on a local filesystem). That is, there is no need to flock() a resource before calling fwrite(); all of the data will be written without interruption.**

第四步，如果我们的日志长度几乎全是小于4096字节，可以退回到第一步的代码，而且还有一个选择，和依次调用 fopen()，fwrite() 以及 fclose() 功能一样。

```
file_put_contents($file, $log, FILE_APPEND);
```

以上四步看似走了一圈冤枉路，但是学习到的经验还是很值得分享的。
