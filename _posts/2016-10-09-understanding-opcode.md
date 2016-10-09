---
layout: post
title:  "深入理解Opcode"
date:   2016-10-09
author: 刘宇
categories: PHP
tags:	php
---
 
本文基于Sara Golemon的Understanding OPcode一文翻译，由于原文写于2008年，很多细节已经发生了较大改变，所以本文基于PHP7进行了改编。
Sara Golemon是一位值得介绍的作者，国内稍微对PHP内核有点研究的程序员，应该都看过@WALU写的《PHP扩展开发及内核应用》，而这本电子书大体上是翻译的Sara Golemon在2005年著作的《Extending and Embedding PHP》，而且这位作者还是一位程序媛。<!-- more -->
 
## What is Opcode
简而言之，它是PHP脚本的编译阶段的中间产物，类似于Java的字节码（bytecode），或者.NET的微软中间语言（MSIL）。举个例子，执行以下PHP代码。

```
<?php
echo "Hello World";
$a = 1 + 1;
echo $a;
```

PHP的Zend内核引擎会经历以下几个步骤

1. **Lexing**：将PHP脚本拆分为语法片段
2. **Parsing**：将语法片段整理为简短而有意义的表达式
3. **Compilation**：将表达式转换为机器指令，也就是本文标题中的Opcode
4. **Execution**：一次一行的执行Opcode，完成PHP脚本的任务

备注：Opcode缓存（Opcache, APC等）可以提升PHP脚本执行的效率，首次执行时经历前面3个步骤，并且把编译生成的Opcode缓存起来，再次执行时只需要经历最后1个步骤。

## How to Lex
学过编译原理的同学都应该对编译原理中的词法分析步骤有所了解，Lex就是一个词法分析的依据表。*Zend/zend_language_scanner.c*会根据*Zend/zend_language_scanner.l*，来对输入的 PHP脚本进行词法分析，从而得到一个一个的词。可以在php.net查看一下*token_get_all*方法介绍，它实际上是Zend内核引擎的词法分析的一个包装。对前面的代码片段执行该方法，可以得到如下输出（为了方便阅读，我做了适当的调整）。

```
array(21) {
  [0]=>array(3) {[0]=>int(379) [1]=>string(6) "<?php\n" [2]=>int(1)}
  [1]=>array(3) {[0]=>int(328) [1]=>string(4) "echo" [2]=>int(2)}
  [2]=>array(3) {[0]=>int(382) [1]=>string(1) " " [2]=>int(2)}
  [3]=>array(3) {[0]=>int(323) [1]=>string(13) ""Hello World"" [2]=>int(2)}
  [4]=>string(1) ";"
  [5]=>array(3) {[0]=>int(382) [1]=>string(1) "\n" [2]=>int(2)}
  [6]=>array(3) {[0]=>int(320) [1]=>string(2) "$a" [2]=>int(3)}
  [7]=>array(3) {[0]=>int(382) [1]=>string(1) " " [2]=>int(3)}
  [8]=>string(1) "="
  [9]=>array(3) {[0]=>int(382) [1]=>string(1) " " [2]=>int(3)}
  [10]=>array(3) {[0]=>int(317) [1]=>string(1) "1" [2]=>int(3)}
  [11]=>array(3) {[0]=>int(382) [1]=>string(1) " " [2]=>int(3)}
  [12]=>string(1) "+"
  [13]=>array(3) {[0]=>int(382) [1]=>string(1) " " [2]=>int(3)}
  [14]=>array(3) {[0]=>int(317) [1]=>string(1) "1" [2]=>int(3)}
  [15]=>string(1) ";"
  [16]=>array(3) {[0]=>int(382) [1]=>string(1) "\n" [2]=>int(3)}
  [17]=>array(3) {[0]=>int(328) [1]=>string(4) "echo" [2]=>int(4)}
  [18]=>array(3) {[0]=>int(382) [1]=>string(1) " " [2]=>int(4)}
  [19]=>array(3) {[0]=>int(320) [1]=>string(2) "$a" [2]=>int(4)}
  [20]=>string(1) ";"
}
```

分析这个输出结果我们可以发现，源码中的操作符都会以字符串的格式原样返回。其他的语句，都会被转换成一个包含三部分的数组，Token ID（使用*token_name*方法可以查询得到易于理解的Token名称），源码中的原来的内容，源码中的行号。很容易就能看出，每个空格都单独拆分出来，可以猜测词法分析是先用操作符，空格或者制表符等分割语句的。

## How to Parse
语法分析先要做的事是去除多余的空格，在剩下的Token集合中，Zend内核引擎查找最基本的表达式。前面的代码片段有3个*statement*，其中1个*statement*可以分割成2个*expression*，$a = 1 + 1，先处理相加，然后处理赋值。全部*expression*列表如下。

1. 打印一个常量字符串
2. 对两个数字相加
3. 将结果赋值给一个变量
4. 打印这个变量

PHP7新增了一个抽象语法树AST的东西，语法分析的输出变成了AST。AST具体是什么，有什么好处，读者可以继续深入学习

https://wiki.php.net/rfc/abstract_syntax_tree。

## How to Compile
然后就改Compile阶段了，它会把AST编译成一个个op_array, 每个op_array包含如下4个部分：

1. Opcode操作类型，比如ECHO，ASSIGN
2. 结果
3. 操作数一
4. 操作数二

我们的PHP代码会被编译成下面这个样子，最后的执行阶段就是一行一行的执行这些机器码。

```
ZEND_ECHO string:Hello World unused unused
ZEND_ASSIGN CV+96 long:2 unknown
ZEND_ECHO CV+96 unused unused
ZEND_RETURN long:1 unused unused
```

\$a已经看不到了，可以简单理解为CV+96就是\$a，其中CV表示编译变量，+96表示偏移量；long：2说明1 + 1的操作已经优化掉了。

备注：由上述优化引申一点，平时写代码时定义的常量，比如1天的时间，不用写86400这种魔术数字了，完全可以写成60 \* 60 \* 24，可读性比较高，也不会额外牺牲性能。

另外，再介绍一个PHP7的opcode查看器

https://github.com/ZaneXie/php7-opdump

Opcode的操作类型，定义在*Zend/zend_vm_opcodes.h*中，数量较多，不在这里列举了。数据类型定义在*Zend/zend_compile.h*中

```
#define IS_CONST        (1<<0)
#define IS_TMP_VAR      (1<<1)
#define IS_VAR          (1<<2)
#define IS_UNUSED       (1<<3)  /* Unused variable */
#define IS_CV           (1<<4)  /* Compiled variable */
#define EXT_TYPE_UNUSED (1<<5)
```
