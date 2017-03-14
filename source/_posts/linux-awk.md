---
layout: post
title: Linux 基础之文本分析 awk
date: 2016-10-28 18:39:04
tags: [Linux]
categories: [Linux]
---

`awk` 是一种模式扫描和处理语言，在对数据进行分析处理时，是十分强大的工具。

```bash
awk [options] 'pattern {action}' file...
```

`awk` 的工作过程是这样的：按行读取输入 (标准输入或文件)，对于符合模式 `pattern` 的行，执行 `actio`n。当 `pattern` 省略时表示匹配任何字符串；当 `action` 省略时表示执行`'{print}'`；它们不可以同时省略。

每一行输入，对 `awk` 来说都是一条记录 (`record`)，`awk` 使用 `$0` 来引用当前记录：

<!-- more -->

```bash
$ head -1 /etc/passwd | awk '{print $0}'
root:x:0:0:root:/root:/bin/bash
```

例子中将命令 `head -1 /etc/passwd` 作为 `awk` 的输入，`awk` 省略了 `pattern`，`action` 为 `print $0`，意为打印当前记录。

对于每条记录，`awk` 使用分隔符将其分割成列，第一列用 `$1` 表示，第二列用 `$2` 表示... 最后一列用 `$NF` 表示

选项 `-F` 表示指定分隔符

如输出文件 `/etc/passwd` 第一行第一列 (用户名) 和最后一列(登录 shell)：

```bash
$ head -1 /etc/passwd | awk -F: '{print $1,$NF}'
root /bin/bash
```

当没有指定分隔符时，使用一到多个 `blank`(空白字符，由空格键或 TAB 键产生) 作为分隔符。输出的分隔符默认为空格。

如输出命令 `ls -l *` 的结果中，文件大小和文件名：


```bash
$ ls -l * | awk '{print $5,$NF}'
13 b.txt
58 c.txt
12 d.txt
0 e.txt
0 f.txt
24 test.sh
$
```

还可以对任意列进行过滤：

```bash
$ ls -l *|awk '$5>20 && $NF ~ /txt$/'
-rw-r--r-- 1 nobody nobody 58 11 月 16 16:34 c.txt
```

其中 `$5>20` 表示第五列的值大于 20；`&&` 表示逻辑与；`$NF ~ /txt$/` 中，`~` 表示匹配，符号 `//` 内部是正则表达式。这里省略了 `action`，整条 awk 语句表示打印文件大小大于 20 字节并且文件名以 txt 结尾的行。

`awk` 用 `NR` 表示行号

```bash
$ awk '/^root/ || NR==2' /etc/passwd
root:x:0:0:root:/root:/bin/bash
bin:x:1:1:bin:/bin:/sbin/nologin
```

例子中 `||` 表示逻辑或，语句表示：输出文件 `/etc/passwd` 中以 root 开头的行或者第二行。

在一些情况下，使用 `awk` 过滤甚至比使用 `grep` 更灵活

如获得 `ifconfig` 的输出中网卡名及其对应的 mtu 值

```bash
$ ifconfig|awk '/^\S/{print $1"\t"$NF}'
ens32:  1500
ens33:  1500
lo:     65536
$
#这里的正则表示不以空白字符开头的行，输出内容中使用 \ t 进行了格式化。
```

以上所说的 `NR`、`NF` 等都是 awk 的内建变量，下面列出部分常用内置变量

```bash
$0          当前记录（这个变量中存放着整个行的内容）
$1~$n       当前记录的第 n 个字段，字段间由 FS 分隔
FS          输入字段分隔符 默认是空格或 Tab
NF          当前记录中的字段个数，就是有多少列
NR          行号，从 1 开始，如果有多个文件话，这个值也不断累加。
FNR         输入文件行号
RS          输入的记录分隔符， 默认为换行符
OFS         输出字段分隔符， 默认也是空格
ORS         输出的记录分隔符，默认为换行符
FILENAME    当前输入文件的名字
```

`awk` 中还可以使用自定义变量，如将网卡名赋值给变量 a，然后输出网卡名及其对应的 `RX bytes` 的值 (注意不同模式匹配及其 action 的写法)：

```bashbash
$ ifconfig|awk '/^\S/{a=$1}/RX p/{print a,$5}'
ens32: 999477100
ens33: 1663197120
lo: 0
```

`awk` 中有两个特殊的 pattern：`BEGIN` 和 `END`；它们不会对输入文本进行匹配，`BEGIN` 对应的 `action` 部分组合成一个代码块，在任何输入开始之前执行；`END` 对应的 `action` 部分组合成一个代码块，在所有输入处理完成之后执行。

```bash
#注意类似于 C 语言的赋值及 print 函数用法
$ ls -l *|awk 'BEGIN{print"size name\n---------"}$5>20{x+=$5;print $5,$NF}END{print"---------\ntotal",x}'
size name
---------
58 c.txt
24 test.sh
---------
total 82
```

`awk` 还支持数组，数组的索引都被视为字符串 (即关联数组)，可以使用 `for` 循环遍历数组元素

如输出文件 `/etc/passwd` 中各种登录 shell 及其总数量

```bash
#注意数组赋值及 for 循环遍历数组的写法
$ awk -F ':' '{a[$NF]++}END{for(i in a) print i,a[i]}' /etc/passwd
/bin/sync 1
/bin/bash 2
/sbin/nologin 19
/sbin/halt 1
/sbin/shutdown 1
```

当然也有 `if` 分支语句

```bash
#注意大括号是如何界定 action 块的
$ netstat -antp|awk '{if($6=="LISTEN"){x++}else{y++}}END{print x,y}'
6 3
```

`pattern` 之间可以用逗号分隔，表示从匹配第一个模式开始直到匹配第二个模式

```bash
$ awk '/^root/,/^adm/' /etc/passwd       
root:x:0:0:root:/root:/bin/bash
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
adm:x:3:4:adm:/var/adm:/sbin/nologin
```

还支持三目操作符 `pattern1 ? pattern2 : pattern3`，表示判断 pattern1 是否匹配，true 则匹配 pattern2，false 则匹配 pattern3，pattern 也可以是类似 C 语言的表达式。

如判断文件 `/etc/passwd` 中 UID 大于 500 的登录 shell 是否为 /bin/bash，是则输出整行，否则输出 UID 为 0 的行：

```bash
#注意为避免混淆对目录分隔符进行了转义
$ awk -F: '$3>500?/\/bin\/bash$/:$3==0 {print $0}' /etc/passwd         
root:x:0:0:root:/root:/bin/bash
learner:x:1000:1000::/home/learner:/bin/bash
#三目运算符也可以嵌套，例子略
```

选项 `-f file` 表示从 file 中读取 awk 指令

```bash
#打印斐波那契数列前十项
$ cat test.awk
BEGIN{
    $1=1
    $2=1
    OFS=","
    for(i=3;i<=10;i++)
    {
        $i=$(i-2)+$(i-1)
    }
    print
}
$ awk -f test.awk
1,1,2,3,5,8,13,21,34,55
```

选项 `-F` 指定列分隔符

```bash
#多个字符作为分隔符时
$ echo 1.2,3:4 5|awk -F '[., :]' '{print $2,$NF}'
2 5
#这里 - F 后单引号中的内容也是正则表达式
```

选项 `-v var=val` 设定变量

```bash
#这里 printf 函数用法类似 C 语言同名函数
$ awk -v n=5 'BEGIN{for(i=0;i<n;i++) printf"%02d\n",i}'  
00
01
02
03
04
```

`print` 等函数还支持使用重定向符 `>` 和 `>>` 将输出保存至文件

```bash
#如按第一列 (IP) 分类拆分文件 access.log，并保存至 ip.txt 文件中
$ awk '{print > $1".txt"}' access.log
$ ls -l 172.20.71.*
-rw-r--r-- 1 root root 5297 11 月 22 21:33 172.20.71.38.txt
-rw-r--r-- 1 root root 1236 11 月 22 21:33 172.20.71.39.txt
-rw-r--r-- 1 root root 4533 11 月 22 21:33 172.20.71.84.txt
-rw-r--r-- 1 root root 2328 11 月 22 21:33 172.20.71.85.txt
```

内建函数

`length()` 获得字符串长度

```bash
$ awk -F: '{if(length($1)>=16)print}' /etc/passwd
systemd-bus-proxy:x:999:997:systemd Bus Proxy:/:/sbin/nologin
```

`split()` 将字符串按分隔符分隔，并保存至数组

```bash
$ head -1 /etc/passwd|awk '{split($0,arr,/:/);for(i=1;i<=length(arr);i++) print arr[i]}'
root
x
0
0
root
/root
/bin/bash
```

`getline` 从输入 (可以是管道、另一个文件或当前文件的下一行) 中获得记录，赋值给变量或重置某些环境变量

```bash
#从 shell 命令 date 中通过管道获得当前的小时数
$ awk 'BEGIN{"date"|getline;split($5,arr,/:/);print arr[1]}'
09
#从文件中获取，此时会覆盖当前的 $0。(注意逐行处理 b.txt 的同时也在逐行从 c.txt 中获得记录并覆盖 $0，当 getline 先遇到 eof 时 <即 c.txt 文件行数较少> 将输出空行)
$ awk '{getline <"c.txt";print $4}' b.txt
"https://segmentfault.com/learnning"
$
#赋值给变量
$ awk '{getline blog <"c.txt";print $0"\n"blog}' b.txt
aasdasdadsad
BLOG ADDRESS IS "https://segmentfault.com/learnning"
$
#读取下一行 (也会覆盖当前 $0)
$ cat file
anny
100
bob
150
cindy
120
$ awk '{getline;total+=$0}END{print total}' file
370
#此时表示只对偶数行进行处理
```

`next` 作用和 getline 类似，也是读取下一行并覆盖 $0，区别是 next 执行后，其后的命令不再执行，而是读取下一行从头再执行。

```bash
#跳过以 a-s 开头的行，统计行数，打印最终结果
$ awk '/^[a-s]/{next}{count++}END{print count}' /etc/passwd
2
$
#又如合并相同列的两个文件
$ cat f.txt
学号 分值
00001 80
00002 75
00003 90
$ cat e.txt
姓名 学号
张三 00001
李四 00002
王五 00003
$ awk 'NR==FNR{a[$1]=$2;next}{print $0,a[$2]}' f.txt e.txt   
姓名 学号 分值
张三 00001 80
李四 00002 75
王五 00003 90
#这里当读第一个文件时 NR==FNR 成立，执行 a[$1]=$2，然后 next 忽略后面的。读取第二个文件时，NR==FNR 不成立，执行后面的打印命令
```

`sub(regex,substr,string)` 替换字符串 string(省略时为 $0) 中首个出现匹配正则 regex 的子串 substr

```bash
$ echo 178278 world|awk 'sub(/[0-9]+/,"hello")'
hello world
$
gsub(regex,substr,string) 与 sub() 类似，但不止替换第一个，而是全局替换

$ head -n5 /etc/passwd|awk '{gsub(/[0-9]+/,"----");print $0}'     
root:x:----:----:root:/root:/bin/bash
bin:x:----:----:bin:/bin:/sbin/nologin
daemon:x:----:----:daemon:/sbin:/sbin/nologin
adm:x:----:----:adm:/var/adm:/sbin/nologin
lp:x:----:----:lp:/var/spool/lpd:/sbin/nologin
```

`substr(str,n,m)` 切割字符串 str，从第 n 个字符开始，切割 m 个。如果 m 省略，则到结尾

```bash
$ echo "hello, 世界！"|awk '{print substr($0,8,1)}'
界
```

`tolower(str)` 和 `toupper(str)` 表示大小写转换

```bash
$ echo "hello, 世界！"|awk '{A=toupper($0);print A}'
HELLO, 世界！
```

`system(cmd)` 执行 shell 命令 cmd，返回执行结果，执行成功为 0，失败为非 0

```bash
#此处 if 语句判断和 C 语言一致，0 为 false，非 0 为 true
$ awk 'BEGIN{if(!system("date>/dev/null"))print"success"}'
success
```

`match(str,regex)` 返回字符串 str 中匹配正则 regex 的位置

```bash
$ awk 'BEGIN{A=match("abc.f.11.12.1.98",/[0-9]{1,3}\./);print A}'
7
```

`awk` 作为一个编程语言可以处理各种各样的问题，甚至于编写应用软件，但它更常用的地方是命令行下的文本分析，生成报表等，这些场景下 awk 工作的很好。工作中如经常有文本分析的需求，那么掌握这个命令的用法将为你节省大量的时间。
