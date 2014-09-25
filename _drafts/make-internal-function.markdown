Make Internal Function

origin 
$(origin <variable>)
返回变量最初定义的位置，有以下返回值：
"undefined" "default" "environment" "file" "command line" "override" "automatic"

example:
    ifeq ("$(origin V)", "command line")
      KBUILD_VERBOSE = $(V)
    endif
    ifndef KBUILD_VERBOSE
      KBUILD_VERBOSE = 0
    endif

    ifeq ($(KBUILD_VERBOSE), 1)
      queit=
      Q=
    else
      quiet=quiet_
      Q=@
    endif

-include filenames 文件不存在，不报错
MAKECMDGOALS make目标, clean, install, distclean　etc.

strip
$(strip <string>)
去除字符串开头和结尾的空格

filter
$(filter <pattern...>, <text>)
以pattern模式过滤text字符串中的单词，返回符合模式的字串。模式之间用空格分隔

$(subst <from>, <to>, <text>)
把字符串中的from替换成to
