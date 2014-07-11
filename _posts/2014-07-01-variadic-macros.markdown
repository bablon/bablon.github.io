---
layout: post
title:  "Variadic Macros"
date:   2014-07-01 14:08:36
categories: c
---

宏可像函数那样使用可变参数定义，GNU C支持两种方式的可变参数宏定义，一种是自身扩展，另一种是C99标准特性。
GNU CPP 扩展支持带命名的可变参数宏，使用方法：

    #define eprintf(format, args...) fprintf(stderr, format, args)
    
C标准方面C99添加可变参数宏特性，使用方法：

    #define eprintf(format, ...) fprintf(stderr, format, __VA_ARGS__)
    
可变参数宏与函数是有区别的，当可变参数为空时，上述两种方式的宏定义在宏展开后会出现语法错误

    eprintf("%s:%d: ")
	    ==>fprintf(stderr, "%s:%d: ",)

使用'##'解决这个问题, 当可变参数为空时，CPP会去除前一个参数后面的逗号

    #define eprintf(format, ...) fprintf(stderr, format, ##__VA_ARGS__)
    #define eprintf(format, args...) fprintf(stderr, format, ##args)

调试宏使用用例：

    #undef DEBUG
    #undef C99_VARIADIC_MACROS
    #undef GNU_VARIDIC_MACROS
    #ifdef DEBUG
    #ifndef C99_VARIADIC_MACROS
    #define debug(fmt, args...) printf(fmt, ##args)
    #else
    #define debug(fmt, ...)	printf(fmt, ##__VA_ARGS__)
    #endif
    #else
    #ifndef C99_VARIADIC_MACROS
    #define debug(fmt, args...)
    #else
    #define debug(fmt, ...)
    #endif
    #endif

