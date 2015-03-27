---
layout: post
title:  "Daylight Save Time"
date:   2015-03-27 22:35:00
categories: nginx
---

你知道夏令时（Daylight Save Time）的概念吗，三天前我是不知道的。

白昼变长的季节，一些国家将时间提前一个小时，鼓励人们早睡早起，通过减少夜晚的活动来节约能源。假如中国在三月最后一个星期到十月最后一个星期的时间范围内实行夏行时，中国的时区将由东八区等效变成东九区。

nginx源码中的时间更新函数nginx_time_update中，有这样一段代码， 我不知道ngx_time_t 结构体中的gmtoff成员及全局变量cached_gmtoff所表示的意思

	ngx_time_tp;

	#if (NGX_HAVE_GETTIMEZONE)
	    tp->gmtoff = ngx_gettimezone();
	    ngx_gmtime(sec + tp->gmtoff * 60, &tm);
	#elif (NGX_HAVE_GMTOFF)
	    ngx_localtime(sec, &tm);
	    cached_gmtoff = (ngx_int_t) (tm.ngx_tm_gmtoff / 60);
	    tp->gmtoff = cached_gmtoff;
	#else
	    ngx_localtime(sec, &tm);
	    cached_gmtoff = ngx_timezone(tm.ngx_tm_isdst);
	    tp->gmtoff = cached_gmtoff;
	#endif

我的Linux环境中将编译最后一个条件中的三行语句，ngx_localtime将日历时间秒数转化成分解的人类可读的日历时间，tm结构体中包括当前时间该地区是否设置夏令时tm.ngx_tim_isdst。宏ngx_timezone定义如下

	#define ngx_timezone(isdst) (- (isdst ? timezone + 3600 : timezone) / 60)

nginx源码中找不到timezone的定义之处，那么它很有可能是在C库与时间有关的头文件中声明的。就像ngx_save_argv接口中，`ngx_os_environ = environ;`语句，你只能找到`extern char **environ;`语句，却找不到`environ`真正声明的源文件，而它则在unistd.h声明。

我开始以为timezone表示地区位于的时区，比如，东十区对应timezone等于10，ngx_timezone宏变得难以理解。实际上timezone表示与世界统一时间UTC相差的秒数，UTC以西为正，以东为负。中国位于东八区，timezone等于-28800。因此cached_gmtoff表示GMT或UTC时间偏移分钟。

C库中依赖时区的时间转换函数会自动调用一个函数tzset来初始化三个全局变量。

	extern char *tzname[2];		/* 时区名 */
	extern long timezone;
	extern int daylight;

如果TZ环境变量没有定义，tzset使用系统时区设置/etc/localtime，它是一个符号链接文件，链接到/usr/share/zoneinfo目录中的一个时区文件。文件格式在tzfile(5)中时间说明。我很想写一个程序来写解析时区文件，但我发现偏离问题太远并没有太大意义。

TZ环境变量有两种格式，流行的格式为TZ=":Asia/Shanghai"，在Linux中冒号可以省略。如果环境变量有设置，tzset从环境变量中解析并设置三个全局变量。如果解析失败使用UTC时间。
