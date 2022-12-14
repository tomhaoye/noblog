---
title: PHP Standard Recommendation
date: 2018-06-16 10:17:53
tags: PHP
categories: 规范
---

非官方规范。

PSR-0(AutoLoading Standard)
---
(14年10月21日起标记为deprecated，由PSR-4代替)

 - 完全合格的命名空间和类名必须有以下结构
 ```
  "\<vendor name>\(<Namespace>)*<Class Name>"
 ```
 - 每个命名空间必须有顶级命名空间
 ```
 ("Vendor Name")
 ```
 - 每个命名空间可以有任意多个子命名空间
 - 每个命名空间在被从文件系统加载时必须被转换为操作系统的路径分隔符
 - 每个"\_"字符在类名中被转换为DIRECTORY_SEPARATOR。"\_"在命名空间中没有明确含义
 - 符合命名标志的命名空间和类名必须以".php"结尾来加载
 - VendorName、命名空间、类名可以由大小写字母组成，其中命名空间和类名是大小写敏感的以保证多系统兼容
 
PSR-1(Basic Coding Standard)
---

 - 源文件必须只使用以下这两种标签
 ```
 <?php 和 <?=
 ```
 - 源文件中php代码的编码格式必须只适用不带BOM的UTF－8（BOM——字节顺序标记）
 - 一个源文件建议只用来做声明（类、函数、常量等）或者只用来做一些引起副作用的操作（如输出信息，修改配置文件等），但不应该同时做这两件事
 - 命名空间和类必须遵守PSR-0
 - 类名必须使用StudlyCaps写法
 - 类中的常量必须只由大写字母和下划线组成
 - 方法名必须使用camelCase写法
 
PSR-2(Coding Style Guide)
---

 - 代码必须遵循PSR-1
 - 代码必须使用4个空格进行缩进，而不是制表符
 - 一行代码的长度不应该有硬限制，软限制为120个字符，建议每行小于80
 - 在命名空间声明下必须空一行，use下同理
 - 类的左、右花括号必须各自成一行
 - 方法的左右花括号都必须各自成一行
 - 所有属性、方法必须有可见性声明；abstract和final必须在可见性声明前，static必须在可见性声明后
 - 在结构控制的关键字后必须空一格；函数调用后面不可有空格
 - 结构控制关键字左花括号必须同一行，右花括号必须放在代码主体下一行
 - 控制结构的左花括号之后不可有空格，右花括号之前也不可有空格
 
PSR-3(Logger Interface)
---

 - LoggerInterface暴露八个接口用来记录八个等级(debug,info,notice,warning,error,critical,alert,emergency)的日志
 - 第九个方法是log，接受日志等级操作为第一个参数。用一个日志等级常量来调用这个方法必须和直接调用指定等级方法的结果一致。用一格本规范中来定义且不为具体实现所知的日志等级来调用该方法必须跑出一个
 ```
 PSR\Log\InvalidArgumentException
 ```
 不推荐使用自定义的日志等级，除非你非常确认当前类库对其支持。
 
PSR-4(Improved AutoLoading)
---
(兼容PSR-0)

 - 术语［类］是一个泛称，它包含类、接口、trait及其他类似结构
 - 完全限定类名应该如下范例
 ```
 <NamespaceName>(<SubNamespace>)*<ClassName>
 ```
 - 
 	+ 完全合规类名必须有一格顶级命名空间
 	+ 完全合规类名可有多个子命名空间
 	+ 完全合规类名应该有一格终止类名
 	+ 下划线在完全合规类名中是没有特殊含义的
 	+ 字母在完全合规类名中可以是任何大小写组合
 	+ 所有类名必须以大小写敏感的方式引用
 - 当完全合规类名载入文件时：
 	+ 在完全合规类名中，连续的一个或几个子命名空间构成的命名空间前缀（不包括顶级命名空间分隔符），至少对应一格基础目录
 	+ 在［命名空间前缀］后的连续子命名空间名称对应一个［基础目录］下的子目录，其中的命名空间分隔符标示目录分隔符。子目录名称必须和子命名空间名大小写匹配
 	+ 终止类名对应一个以.php结尾的文件。文件名必须和终止类名大小写匹配
 - 自动载入器的实现不可以抛出任何异常，不可以引发任何等级的错误，也不应该有返回值

