---
layout: post
title: Linux 基础之语法
date: 2016-10-28 18:39:04
tags: [Linux]
categories: [Linux]
---


本文开始正式介绍 shell 脚本的编写方法以及 bash 的语法。

## 定义

`元字符` 用来分隔词 (token) 的单个字符，包括：

```
| & ; () <> space tab
```

<!-- more -->

`token` 是指被 shell 看成一个单一单元的字符序列

bash 中包含三种基本的 token：`保留关键字`，`操作符`，`单词`。

`保留关键字`是指在 shell 中有明确含义的词语，通常用来表达程序控制结构。包括：

```
! case coproc do done elif else esac fi for function if in select then until while {} time [[]]
```

`操作符`由一个或多个`元字符`组成，其中`控制操作符`包括：

```
|| & && ; ;; () | |& <newline>
```

余下的 shell 输入都可以视为普通的`单词`(word)。

shell 脚本是指包含若干 shell 命令的文本文件，标准的 bash 脚本的第一行形如 `#!/bin/bash`，其中顶格写的字符 #! 向操作系统申明此文件是一个脚本，紧随其后的 `/bin/bash` 是此脚本程序的解释器，解释器可以带一个选项 (选项一般是为了对一些情况做特殊处理，比如 `-x` 表示开启 bash 的调试模式)。

除首行外，其余行中以符号 `#` 开头的单词及本行中此单词之后的字符将作为注释，被解析器所忽略。

## 语法

相比于其他更正式的语言，bash 的语法较为简单。大多数使用 bash 的人员，一般都先拥有其他语言的语法基础，在接触 bash 的语法之后，会自然的将原有语法习惯套用到 bash 中来。事实上，bash 的语法灵活多变，许多看起来像是固定格式的地方，实际上并不是。这让一些初学者觉得 bash 语法混乱不堪，复杂难记。这和 bash 的目的和使用者使用 bash 的目的有很大的关系，bash 本身是为了提供一个接口，来支持用户通过命令与操作系统进行交互。用户使用 bash，一般是为了完成某种系统管理的任务，而不是为了做一款独立的软件。这些，都使人难以像学习其他编程语言那样对 bash 认真对待。其实，只要系统学习一遍 bash 语法以及一条命令的执行流程，就可以说掌握了 bash 脚本编程的绝大多数内容。

bash 语法只包括六种：`简单命令`、`管道命令`、`序列命令`、`复合命令`、`协进程命令` (bash 版本 4.0 及以上) 和`函数定义`。

### 简单命令

shell 简单命令 (`Simple Commands`) 包括命令名称，可选数目的参数和重定向(`redirection`)。我们在 Linux 基础命令介绍系列里所使用的绝大多数命令都是简单命令。另外，在命令名称前也可以有若干个变量赋值语句(如上一篇所述，这些变量赋值将作为命令的临时环境变量被使用，后面有例子)。简单命令以上述`控制操作符`为结尾。

shell 命令执行后均有`返回值` (会赋给特殊变量 `$?`)，是范围为 0-255 的数字。返回值为 0，表示命令执行成功；非 0，表示命令执行失败。(可以使用命令 `echo $?` 来查看前一条命令是否执行成功)

### 管道命令

管道命令 (`pipeline`) 是指被 `|` 或 `|&` 分隔的一到多个命令。格式为：

```
[time [-p]] [ ! ] command1 [ | command2 ... ]
```

其中保留关键字 `time` 作用于管道命令表示当命令执行完成后输出消耗时间 (包括用户态和内核态占用时间)，选项 `-p` 可以指定时间格式。

默认情况下，管道命令的返回值是最后一个命令的返回值，为 0，表示 `true`，非 0，则表示 `false`；当保留关键字 `!` 作用于管道命令时，会对管道命令的返回值进行取反。

之前我们介绍过管道的基本用法，表示将 `command1` 的标准输出通过管道连接至 `command2` 的标准输入，这个连接要先于命令的其他重定向操作 (试对比 `>/dev/null 2>&1` 和 `2>&1 >/dev/null` 的区别)。如果使用 `|&`，则表示将 `command1` 的标准输出和标准错误都连接至管道。

管道两侧的命令均在子 shell(`subshell`) 中执行，这里需要注意：在子 shell 中对变量进行赋值时，父 shell 是不可见的。

```
#例如
[root@centos7 ~]# echo 12345|read NUM
[root@centos7 ~]# echo $NUM
                    #由于 echo 和 read 命令均在子 shell 中执行，所以当执行完毕时，在父 shell 中输出变量的值为空
[root@centos7 ~]#
```

### 序列命令

序列命令 (`list`) 是指被控制操作符`;,&,&&` 或 `||` 分隔的一到多个管道命令，以`;`、`&` 或 `<newline>` 为结束。

在这些控制操作符中，`&&` 和 `||` 有相同的优先级，然后是 `;` 和 `&`(也是相同的优先级)。

如果命令以 `&` 为结尾，此命令会在一个子 shell 中后台执行，当前 shell 不会等待此命令执行结束，并且不论它是否执行成功，其返回值均为 0。

以符号 `;` 分隔的命令按顺序执行 (和换行符的作用几乎相同)，shell 等待每个命令执行完成，它们的返回值是最后一个命令的返回值。

以符号 `&&` 和 `||` 连接的两个命令存在逻辑关系。

`command1 && command2`：先执行 command1，当且仅当 command1 的返回值为 0，才执行 command2。

`command1 || command2`：先执行 command1，当且仅当 command1 的返回值非 0，才执行 command2。

脚本举例：

```
#!/bin/bash
#简单命令
echo $PATH > file
#管道命令
cat file|tr ':' ' '
#序列命令
IFS=':' read -a ARRAY <file && echo ${ARRAY[4]} || echo "赋值失败"
echo "命令返回值为：$?。"
#验证变量的临时作用域
echo "$IFS"|sed 'N;s/[\t\n]/-/g'
```

执行结果 (在脚本所在目录直接执行./test.sh)：

```
[root@centos7 ~]# ./test.sh   
/usr/local/sbin /usr/local/bin /usr/sbin /usr/bin /root/bin
/root/bin
命令返回值为：0。
```

注意例子中序列命令的写法，其中 IFS=':'只临时对内置命令 read 起作用 (作为单词分隔符来分隔 read 的输入)，read 命令结束后，IFS 又恢复到原来的值：$'\d\n'。
&& 和 || 在这里类似于分支语句，read 命令执行成功则执行输出数组的第五个元素，否则执行输出 "赋值失败"。

### 复合命令

#### 1、(list)

list 将在 subshell 中执行 (注意赋值语句和内置命令修改 shell 状态不能影响当父 shell)，返回值是 list 的返回值。

此复合命令前如果使用扩展符 $，shell 称之为命令替换 (另一种写法为 \\list\`)。shell 会把命令的输出作为命令替换 \` 扩展之后的结果使用。

命令替换可以嵌套。

### 2、{list;}

list 将在当前 shell 环境中执行，必须以换行或分号为结尾 (即使只有一个命令)。注意不同于 shell 元字符：(和)，{和} 是 shell 的保留关键字，因为保留关键字不能分隔单词，所以它们和 list 之间必须有空白字符或其他 shell 元字符。

### 3、((expression))

expression 是数学表达式 (类似 C 语言的数学表达式)，如果表达式的值非 0，则此复合命令的返回值为 0；如果表达式的值为 0，则此复合命令的返回值为 1。

此种复合命令和使用内置命令 let "expression" 是一样的。

数学表达式中支持如下操作符，操作符的优先级，结合性，计算方法都和 C 语言一致 (按优先级从上到下递减排列)：

```bash
id++ id--      # 变量后自加 后自减
++id --id       # 变量前自加 前自减
- +             # 一元减 一元加
! ~             # 逻辑否定 位否定
**              # 乘方
* / %           # 乘 除 取余
+ -             # 加 减
<<>>           # 左位移 右位移
<=>= < >       # 比较大小
== !=           # 等于 不等于
&               # 按位与
^               # 按位异或
|               # 按位或
&&              # 逻辑与
||              # 逻辑或
expr?expr:expr  # 条件表达式
= *= /= %= += -= <<=>>= &= ^= |=   # 赋值表达式
expr1 , expr2   # 逗号表达式
```

在数学表达式中，可以使用变量作为操作数，变量扩展要先于表达式的求值。变量还可以省略扩展符号 $，如果变量的值为空或非数字和运算符的其他字符串，将使用 0 代替它的值做数学运算。

以 0 开头的数字将被解释为八进制数，以 0x 或 0X 开头的数字将被解释为十六进制数。其他情况下，数字的格式可以是 [base#]n。可选的 base# 表示后面的数字 n 是以 base(范围是 2-64) 为基的数字，如 2#11011 表示 11011 是一个二进制数字，命令 ((2#11011)) 的作用会使二进制数转化为十进制数。如果 base# 省略，则表示数字以 10 为基。

复合命令 ((expression)) 并不会输出表达式的结果，如果需要得到结果，需使用扩展符 $ 表示数学扩展(另一种写法为 $[expression])。数学扩展也可以嵌套。

括号 () 可以改变表达式的优先级。

脚本举例：

```bash
#!/bin/bash
# (list)
(ls|wc -l)
#命令替换并赋值给数组 注意区分数组赋值 array=(...) 和命令替换 $(...)
array=($(seq 10 10 $(ls|wc -l) | sed -z 's/\n/ /g'))
#数组取值
echo "${array[*]}"
# {list;}
#将文件 file1 中的第一行写入 file2，{list;} 是一个整体。
{read line;echo $line;} >file2 <file1
#数学扩展
A=$(wc -c file2 |cut -b1)
#此时变量 A 的值为 5
B=4
echo $((A+B))
echo $(((A*B)**2))
#赋值并输出
echo $((A|=$B))
#条件运算符 此命令意为：判断表达式 A>=7 是否为真，如果为真则计算 A-1，否则计算 (B<<1)+3。然后将返回结果与 A 作异或运算并赋值给 A。
((A^=A>=7?A-1:(B<<1)+3))
echo $A
```

执行结果：

```bash
[root@centos7 temp]# ./test.sh
43
10 20 30 40
9
400
5
14
```

### 4、[[expression]]

此处的 expression 是条件表达式 (并非数学扩展中的条件表达式)。此种命令的返回值取决于条件表达式的结果，结果为 true，则返回值为 0，结果为 false，则返回值为 1。

条件表达式除可以用在复合命令中外，还可以用于内置命令 test 和 [，由于 test、[[、]]、[和] 是内置命令或保留关键字，所以同保留关键字 {和} 一样，它们与表达式之间都要有空格或其他 shell 元字符。

条件表达式的格式包括：

```bash
-b file             #判断文件是否为块设备文件
-c file             #判断文件是否为字符设备文件
-d file             #判断文件是否为目录
-e file             #判断文件是否存在
-f file             #判断文件是否为普通文件
-g file             #判断文件是否设置了 SGID
-h file             #判断文件是否为符号链接
-p file             #判断文件是否为命名管道文件
-r file             #判断文件是否可读
-s file             #判断文件是否存在且内容不为空 (也可以是目录)
-t fd               #判断文件描述符 fd 是否开启且指向终端
-u file             #判断文件是否设置 SUID
-w file             #判断文件是否可写
-x file             #判断文件是否可执行
-S file             #判断文件是否为 socket 文件
file1 -nt file2     #判断文件 file1 是否比 file2 更新 (根据 mtime)，或者判断 file1 存在但 file2 不存在
file1 -ot file2     #判断文件 file1 是否比 file2 更旧，或者判断 file2 存在但 file1 不存在
file1 -ef file2     #判断文件 file1 和 file2 是否互为硬链接
-v name             #判断变量状态是否为 set(见上一篇)
-z string           #判断字符串是否为空
-n string           #判断字符串是否非空
string1 == string2  #判断字符串是否相等
string1 = string2   #判断字符串是否相等
string1 != string2  #判断字符串是否不相等
string1 <string2   #判断字符串 string1 是否小于字符串 string2(字典排序)，用于内置命令 test 中时，小于号需要转义：\<
string1 > string2   #判断字符串 string1 是否大于字符串 string2(字典排序)，用于内置命令 test 中时，大于号需要转义：\>
NUM1 -eq NUM2       #判断数字是否相等
NUM1 -ne NUM2       #判断数字是否不相等
NUM1 -lt NUM2       #判断数字 NUM1 是否小于数字 NUM2
NUM1 -le NUM2       #判断数字 NUM1 是否小于等于数字 NUM2
NUM1 -gt NUM2       #判断数字 NUM1 是否大于数字 NUM2
NUM1 -ge NUM2       #判断数字 NUM1 是否大于等于数字 NUM2
[[expr]]和 [ expr ](test expr 是[ expr ] 的另一种写法，效果相同)还接受如下操作符(从上到下优先级递减)：

! expr    #表示对表达式 expr 取反
(expr)  #表示提高 expr 的优先级
expr1 -a expr2  #表示对两个表达式进行逻辑与操作，只能用于 [expr] 和 test expr 中
expr1 && expr2  #表示对两个表达式进行逻辑与操作，只能用于 [[expr]] 中
expr1 -o expr2  #表示对两个表达式进行逻辑或操作，只能用于 [expr] 和 test expr 中
expr1 || expr2  #表示对两个表达式进行逻辑或操作，只能用于 [[expr]] 中
在使用操作符 == 和!= 判断字符串是否相等时，在 [[expr]] 中等号右边的 string2 可以被视为模式匹配 string1，规则和通配符匹配一致。([ expr ]不支持)
```

[[expr]] 中比较两个字符串时还可以用操作符 =~，符号右边的 string2 可以被视为是正则表达式匹配 string1，如果匹配，返回真，否则返回假。

### 5、if list; then list; [elif list; then list;] ... [ else list; ] fi

条件分支命令。首先判断 if 后面的 list 的返回值，如果为 0，则执行 then 后面的 list；如果非 0，则继续判断 elif 后面的 list 的返回值，如果为 0，则......，若返回值均非 0，则最终执行 else 后的 list。fi 是条件分支的结束词。

注意这里的 list 均是命令，由于要判断返回值，通常使用上述条件表达式来进行判断

形如：

```bash
if [expr]
then
    list
elif [expr]
then
    list
...
else
    list
fi
```

甚至，许多人认为这样就是 if 语句的固定格式，其实 if 后面可以是任何 shell 命令，只要能够判断此命令的返回值。如：

```bash
[root@centos7 ~]# if bash;then echo true;else echo false;fi
[root@centos7 ~]#   #执行后没有任何输出
[root@centos7 ~]# exit
exit
true                #由于执行了 bash 命令开启了一个新的 shell，所以执行 exit 之后 if 语句才获得返回值，并做了判断和输出
[root@centos7 ~]#
```

脚本举例：

```bash
#!/bin/bash
#条件表达式
declare A
#判断变量 A 是否 set
[[-v A]] && echo "var A is set" || echo "var A is unset"
#判断变量 A 的值是否为空
[! $A] && echo false || echo true
test -z $A && echo "var A is empty"
#通配与正则
A="1234567890abcdeABCDE"
B='[0-9]*'
C='[0-9]{10}\w+'
[[$A = $B]] && echo '变量 A 匹配通配符 [0-9]*' || echo '变量 A 不匹配通配符 [0-9]*'
[$A == $B] && echo '[ expr ] 中能够使用通配符' || echo '[ expr ] 中不能使用通配符'
[[$A =~ $C]] && echo '变量 A 匹配正则 [0-9]{10}\w+' || echo '变量 A 不匹配正则 [0-9]{10}\w+'
#if 语句
# 此例并没有什么特殊的意义，只为说明几点需要注意的地方：
# 1、if 后面可以是任何能够判断返回值的命令
# 2、直接执行复合命令 ((...)) 没有输出，要取得表达式的值必须通过数学扩展 $((...))
# 3、复合命令 ((...)) 中表达式的值非 0，返回值才是 0
number=1
if  if test -n $A
    then
        ((number+1))
    else
        ((number-1))
    fi
then
    echo "数学表达式值非 0，返回值为 0"
else
    echo "数学表达式值为 0，返回值非 0"
fi
# if 语句和控制操作符 && || 连接的命令非常相似，但要注意它们之间细微的差别：
# if 语句中 then 后面的命令不会影响 else 后的命令的执行
# 但 && 后的命令会影响 || 后的命令的执行
echo '---------------'
if [[-r file && ! -d file]];then
    grep -q hello file
else
    awk '/world/' file
fi
echo '---------------'
# 上面的 if 语句无输出，但下面的命令有输出
[-r file -a ! -d file] && grep -q hello file || awk '/world/' file
# 可以将控制操作符连接的命令写成这样来忽略 && 后命令的影响 (使用了内置命令 true 来返回真):
echo '---------------'
[-r file -a ! -d file] && (grep -q hello file;true) || awk '/world/' file
```

### 6、for name [[in [ word ...] ];]do list;done

### 7、for ((expr1;expr2;expr3));do list;done

bash 中的 for 循环语句支持如上两种格式，在第一种格式中，先将 in 后面的 word 进行扩展，然后将得到的单词列表逐一赋值给变量 name，每一次赋值都执行一次 do 后面的 list，直到列表为空。如果 in word 被省略，则将位置变量逐一赋值给 name 并执行 list。第二种格式中，双圆括号内都是数学表达式，先计算 expr1，然后反复计算 expr2，直到其值为 0。每一次计算 expr2 得到非 0 值，执行 do 后面的 list 和第三个表达式 expr3。如果任何一个表达式省略，则表示其值为 1。for 语句的返回值是执行最后一个 list 的返回值。

脚本举例：

```bash
#!/bin/bash
# word 举例
for i in ${a:=3} $(head -1 /etc/passwd) $((a+=2))
do
    echo -n "$i"
done
echo $a
# 省略 in word
declare -a array
for number
do
    array+=($number)
done
echo ${array[@]}
# 数学表达式格式
for((i=0;i<${#array[*]};i++))
do
    echo -n "${array[$i]}"|sed 'y/1234567890/abcdefghij/'
done;echo
```

执行：

```bash
[root@centos7 temp]# ./test.sh "$(seq 10)"   # 注意此处 "$(seq 10)" 将作为一个整体赋值给 $1，如果去掉双引号将会扩展成 10 个值并赋给 $1 $2 ... ${10}
3 root:x:0:0:root:/root:/bin/bash 5 5        # 是否带双引号并不影响执行结果，只影响第二个 for 语句的循环次数。
1 2 3 4 5 6 7 8 9 10
a b c d e f g h i aj
[root@centos7 temp]#
```

### 8、while list-1; do list-2; done

### 9、until list-1; do list-2; done

while 命令会重复执行 list-2，只要 list-1 的返回值为 0；until 命令会重复执行 list-2，只要 list-1 的返回值为非 0。while 和 until 命令的返回值是最后一次执行 list-2 的返回值。

break 和 continue 两个内置命令可以用于 for、while、until 循环中，分别表示跳出循环和停止本次循环开始下一次循环。

### 10、case word in [[(] pattern [ | pattern]...) list ;;] ... esac

case 命令会将 word 扩展后的值和 in 后面的多个不同的 pattern 进行匹配 (通配符匹配)，如果匹配成功则执行相应的 list。

list 后使用操作符;; 时，表示如果执行了本次的 list，那么将不再进行下一次的匹配，case 命令结束；

使用操作符;&，则表示执行完本次 list 后，再执行紧随其后的下一个 list(不判断是否匹配)；

使用操作符;;&，则表示继续下一次的匹配，如果匹配成功，那么执行相应的 list。

case 命令的返回值是执行最后一个命令的返回值，当匹配均没有成功时，返回值为 0。

脚本举例：


```bash
#!/bin/bash
# while
unset i j
while ((i++<$(grep -c '^processor' /proc/cpuinfo)))
do
    #每个后台运行的 yes 命令将占满一核 CPU
    yes >/dev/null &
done
# -------------------------------------------------
# until
# 获取 yes 进程 PID 数组
PIDS=($(ps -eo pid,comm|grep -oP '\d+(?= yes$)'))
# 逐个杀掉 yes 进程
until ! ((${#PIDS[*]}-j++))
do
    kill ${PIDS[$j-1]}
done
# -------------------------------------------------
# case
user_define_command &>/dev/null
case $? in
0) echo "执行成功" ;;
1) echo "未知错误" ;;
2) echo "误用 shell 命令" ;;
126) echo "权限不够" ;;
127) echo "未找到命令" ;;
130) echo "CTRL+C 终止" ;;
*) echo "其他错误" ;;
esac
# -------------------------------------------------
#定义数组
c=(1 2 3 4 5)
#关于各种复合命令结合使用的例子：
echo -e "$(
for i in ${c[@]}
do
    case $i in
    (1|2|3)
        printf "%d\n" $((i+1))
        ;;
    (4|5)
        printf "%d\n" $((i-1))
        ;;
    esac
done
)" | while read NUM
do
    if [[$NUM -ge 4]];then
        printf "%s\n" "数字 ${NUM} 大于等于 4"
    else
        printf "%s\n" "数字 ${NUM} 小于 4"
    fi
done
```

执行结果：

```bash
[root@centos7 temp]# ./test.sh
./test.sh: 行 18: 18671 已终止               yes > /dev/null
./test.sh: 行 18: 18673 已终止               yes > /dev/null
./test.sh: 行 18: 18675 已终止               yes > /dev/null
./test.sh: 行 18: 18677 已终止               yes > /dev/null
./test.sh: 行 18: 18679 已终止               yes > /dev/null
./test.sh: 行 18: 18681 已终止               yes > /dev/null
./test.sh: 行 20: 18683 已终止               yes > /dev/null
./test.sh: 行 20: 18685 已终止               yes > /dev/null
未找到命令
数字 2 小于 4
数字 3 小于 4
数字 4 大于等于 4
数字 3 小于 4
数字 4 大于等于 4
[root@centos7 temp]#
```

### 11、select name [in word] ; do list ; done

select 命令适用于交互式菜单选择场景。word 的扩展结果组成一系列可选项供用户选择，用户通过键入提示字符中可选项前的数字来选择特定项目，然后执行 list，完成后继续下一轮选择，需要使用内置命令 break 来跳出循环。

脚本举例：

```bash
#!/bin/bash
echo "系统信息："
select item in "host_name" "user_name" "shell_name" "quit"
do
    case $item in
     host*) hostname;;
     user*) echo $USER;;
     shell*) echo $SHELL;;
     quit) break;;
    esac
done
```

执行结果：

```bash
[root@centos7 ~]# ./test.sh
系统信息：
1) host_name
2) user_name
3) shell_name
4) quit
#? 1
centos7
#? 2
root
#? 3
/bin/bash
#? 4
[root@centos7 ~]#
```

## 协进程命令

协进程命令是指由保留关键字 coproc 执行的命令 (bash4.0 版本以上)，其命令格式为：

```bash
coproc [NAME] command [redirections]
```

命令 command 在子 shell 中异步执行，就像被控制操作符 & 作用而放到了后台执行，同时建立起一个双向管道，连接该命令和当前 shell。

执行此命令，即创建了一个协进程，如果 NAME 省略 (command 为简单命令时必须省略，此时使用默认名 COPROC)，则称为匿名协进程，否则称为命名协进程。

此命令执行时，command 的标准输出和标准输入通过双向管道分别连接到当前 shell 的两个文件描述符，然后文件描述符又分别赋值给了数组元素 NAME[0] 和 NAME[1]。此双向管道的建立要早于命令 command 的其他重定向操作。被连接的文件描述符可以当成变量来使用。子 shell 的 pid 可以通过变量 NAME_PID 来获得。

关于协进程的例子，我们在下一篇给出。

## 函数定义

bash 函数定义的格式有两种：

```bash
name () compound-command [redirection]
function name [()] compound-command [redirection]
```

这样定义了名为 name 的函数，使用保留关键字 function 定义函数时，括号可以省略。函数的代码块可以是任意一个上述的复合命令 (compound-command)。

脚本举例：

```bash
#!/bin/bash
#常用定义方法：
func_1() {
    #局部变量
    local num=6
    #嵌套执行函数
    func_2
    #函数的 return 值保存在特殊变量? 中
    if [$? -gt 10];then
        echo "大于 10"
    else
        echo "小于等于 10"
    fi
}
################
func_2()
{
    # 内置命令 return 使函数退出，并使其的返回值为命令后的数字
    # 如果 return 后没有参数，则返回函数中最后一个命令的返回值
    return $((num+5))
}
#执行。就如同执行一个简单命令。函数必须先定义后执行 (包括嵌套执行的函数)
func_1
###############
#一般定义方法
#函数名后面可以是任何复合命令：
func_3() for NUM
do
    # 内置命令 shift 将会调整位置变量，每次执行都把前 n 个参数撤销，后面的参数前移。
    # 如果 shift 后的数字省略，则表示撤销第一个参数 $1，其后参数前移 ($2 变为 $1....)
    shift
    echo -n "$((NUM+$#))"
done
#函数内部位置变量被重置为函数的参数
func_3 `seq 10`;echo
```

执行结果：

```bash
[root@centos7 temp]# ./test.sh   
大于 10
10 10 10 10 10 10 10 10 10 10
[root@centos7 temp]#
```

这些就是bash的所有命令语法。bash中任何复杂难懂的语句都是这些命令的变化组合。
