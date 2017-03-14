---
layout: post
title: Linux 基础之变量
date: 2016-10-28 18:39:04
tags: [Linux, shell]
categories: [Linux]
---


本篇开始，介绍 shell 脚本编程，更确切的说是 bash 脚本编程 (版本：4.2.46(1)-release)。我们从变量开始。

和所有的编程语言一样，bash 也提供变量，变量是一些用来指代数据并支持数据操作的名称。

## 类型

### 环境变量

#### 概念

当我们通过 ssh 等工具登录系统时，便获得一个 shell(一个 bash 进程)，bash 在启动过程中会加载一系列的配置文件，这些配置文件的作用就是为用户准备好 bash 环境，大部分 `环境变量` 都是在这些文件中被设置的。

<!-- more -->

`登录 shell`(login shell) 是指需要通过输入用户名、密码登录之后获得的 shell(或者通过选项 "`--login`" 生成的 shell)。登录 shell 的进程名为 `-bash`，非登录 shell(比如在桌面环境下通过打开一个 "`终端`" 窗口程序而获得的 shell) 的进程名为 `bash`。

```
$ ps -ef|grep [b]ash
root      2917  2915  0 14:25 pts/3    00:00:00 -bash
root      2955  2953  0 14:25 pts/5    00:00:00 -bash
root      3070  3068  0 14:42 pts/4    00:00:00 -bash
```

`交互式 shell`(interactive shell) 是指 shell 与用户进行交互，shell 需要等待用户的输入 (键入一条命令后并按下回车键)，用户需要等待命令的执行和输出。当把一到多个命令写入一个文件，并通过执行这个文件来执行这些命令时，bash 也会为这些命令初始化一个 shell 环境，这样的 shell 称为`非交互式 shell`。

环境变量 - 中存储了当前 shell 的选项标志，其中如果包含字符 `i` 则表示此 shell 是交互式 shell：

```
#输出变量'-'的值
$ echo $-
himBH
```

通常，一个`登录 shell`(包括交互式登录 shell 和使用 "--login" 选项的非交互 shell) 首先读取并执行文件 `/etc/profile`(此文件会在结尾处判断并执行 `/etc/profile.d/` 中所有以. sh 结尾的文件)；然后按顺序搜索用户家目录下的`~/.bash_profile`、`~/.bash_login` 和`~/.profile`，并执行找到的第一个可读文件 (在 centos7 系统中是文件`~/.bash_profile`，此文件会进一步判断并执行文件~/.bashrc，然后再进一步判断并执行文件 `/etc/bashrc`)。当一个登录 shell 登出时 (exit)，会执行文件`~/.bash_logout` 和 `/etc/bash.bash_logout`(如果文件存在的话)。

`交互式非登录 shell` 启动时，bash 会读取并执行文件`~/.bashrc`。

`非交互式 shell` 启动时 (如脚本中)，会继承派生出此 shell 的父 shell 的环境变量并执行环境变量 `BASH_ENV` 的值中所指代的文件。

#### 作用

环境变量的作用主要是影响 shell 的行为，在整个 bash 进程的生命周期中，会多次使用到环境变量。每个由当前 bash 进程派生出的子进程 (包括子 shell)，都会继承当前 bash 的环境变量 (除非子进程对继承的环境变量进行了重新赋值，否则它们的值将和父进程相同)。下面列出部分常用环境变量及其作用：

`PATH` 其值是一个以冒号分隔的目录列表，定义了 shell 命令的搜索路径。

```
$ echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/root/bin
```

`PS1` 首要命令提示符。

```
#笔者环境下变量 PS1 的值：
$ echo $PS1
[\u@\h \W]\$
# \u 表示当前用户的用户名
# \h 表示主机名字符串中直到第一个. 之前的字符
# \W 表示当前路径的 basename，用户家目录会被缩写为波浪号 (~)
# \$ 如果用户 UID 为 0，则为符号 #，否则为符号 $
```

`PS2` 连续性 交互式提示符。当输入的命令分为好几行时，新行前出现的字符串即为 PS2 变量的值。

```
$ echo $PS2
>
```

`PS3` shell 脚本中 select 关键字提示符

`PS4` shell 调试模式下的提示符

`HOME` 当前用户的家目录

`PWD` 当前工作目录

`OLDPWD` 前一个工作目录

```
# cd 命令后如果没有任何参数时，则使用 $HOME 作为默认参数
$ cd
$  
# cd 命令后的参数 - 等同于 $OLDPWD
$ cd -
/root/temp/tmp
# cd 命令的成功执行会更新两个环境变量 (PWD 和 OLDPWD) 的值
$ echo $PWD $OLDPWD
/root/temp/tmp /root
```

`RANDOM` 每次引用此变量，都会生成一个 0 到 32767 之间的随机数字

`BASH_VERSION` 其值为当前 bash 版本号

```
$ echo $BASH_VERSION
4.2.46(1)-release
```

`IFS` 域分隔符，用来分隔单词。默认值为 空格键 TAB 键 回车键产生的字符

```
#可以用 set 命令查看当前环境下的所有变量
$ set|grep IFS
IFS=$'\t\n'
```

本系列中在涉及到具体环境变量的时候还有更详细的解释和用法描述。

### 自定义变量

#### 普通变量

bash 除了在初始化时自动设置的变量外，用户还可以根据需要手动设置变量。普通变量赋值语句写法：

```
name=[value]
```

其中 `name` 为变量名，变量名必须以英文字母 (`[a-zA-Z]`) 或下划线 (`_`) 开头，其余字符可以是英文字母、下划线或数字 (`[0-9]`)。变量名是大小写敏感的。在给变量赋值时，等号两边不能有任何空白字符。等号后的值(`value`) 可以省略，如果省略，则变量的值为空字符串(`null`)。

#### 数组变量

bash 提供一维的索引和关联数组变量，`索引数组`是以数字为下标的数组，`关联数组`是以字符串为下标的数组 (类似其他语言中的 map 或 dict)。

数组赋值语句写法：

```
name=(value1 value2 ... valueN)
```

其中每一个 `value` 都是类似以 `[subscript]=string` 的格式，索引数组赋值时可以省略 `[subscript]=`，关联数组不能省略。

```bash
#索引数组赋值的一般形式：
name_index=(aa bb cc dd ee)
#关联数组赋值之前，必须先通过内置命令 declare 进行声明，然后才能赋值：
declare -A name_associate
name_associate=([aa]=1 [bb]=2 [cc]=3 [dd]=4)
```

所谓`内置命令`，是指由 bash 自身实现的命令，它们的执行就相当于执行 bash 的一个函数，并不需要派生出新的子进程。

`外部命令`是指那些不是由 bash 自身实现的命令 (如环境变量 PATH 目录内的命令)。原则上所有命令都应该外部实现 (避免臃肿及和其他系统耦合度过高)，但是，外部命令的执行，意味着创建子进程，而子进程对环境变量等的更改是无法影响父进程的。bash 想要更改自身的一些状态时，就得靠`内置命令`来实现。例如，改变工作目录命令 `cd`，就是一个典型的例子 (cd 命令会更改当前所处目录，并更新环境变量 `PWD` 和 `OLDPWD`，如果此功能由外部实现，更改目录的目的就无法实现了)。

### 特殊变量

bash 中还支持一些表示特殊意义的变量，这些变量不能使用上述语句进行赋值。

```bash
$0 本程序所处的进程名。
$n n 是从 1 开始的整数，表示当前进程参数，$1 表示第一个参数、$2 表示第二个参数...$n 表示第 n 个参数。如果 n 大于 10，取值时需要写成 ${n} 的格式。当执行函数时，这些位置变量被临时替换为函数的第一个参数、第二个参数、、、第 N 个参数。
$* 表示当前进程的所有参数。$1 $2 ... ${n}。当处于双引号中取值时，所有结果被当成一个整体，即 "$*" 等同于 "$1 $2 ... ${n}"。
$@ 表示当前进程的所有参数。$1 $2 ... ${n}。当处于双引号中取值时，每个结果被当成单独的单词，即 "$@" 等同于 "$1" "$2" ... "${n}"。
$# 表示当前进程的参数个数。
$? 表示前一个命令的返回码，为 0 表示前一个命令执行成功，非 0 表示执行失败。
$- 表示当前 shell 的选项标志。
$$ 表示当前 shell 的 PID。
$! 表示最近一次执行的后台命令的 PID。
$_ 在 shell 初始启动时表示启动此 shell 命令的绝对路径或脚本名，随后，表示前一条命令的最后一个参数。
```

## 声明/定义及赋值

通常 bash 的变量是不需要提前声明的，可以直接进行赋值。变量的值均被视为`字符串` (在一些情况下也可以视为数字)。当对变量有特殊需要时，也可以先声明变量 (如前面关联数组的声明)。

bash 提供了几个和变量声明及赋值相关的内置命令，这些命令即可以和赋值语句写在同一行 (表示声明及赋值)，也可以只跟变量名 (表示声明)。

`[]` 表示可选：

```bash
declare [options] name[=value] ...
typeset [options] name[=value] ...
```

这是两个起同样作用的命令，用来声明变量；

```bash
#如声明一个普通变量：
declare name[=value]
#如声明一个只能存储数字的变量：
declare -i name[=value]
#选项 - i 表示为变量增加一个数字属性，变量 name 中只能存储数字，如果将字符串赋给此变量时，变量的值为 0
#如声明一个索引数组
declare -a name_index[=(aa bb cc dd ee)]
#如声明一个变量，并将其导出为环境变量
declare -x name[=value]
#如声明一个只读变量
declare -r name[=value]
```

以上选项可以使用命令 `declare +OPTION name` 撤销变量 name 的属性 (只读变量除外)
内置命令 `export` 作用于赋值语句时，和 `declare -x` 类似表示导出变量为环境变量 (临时有效，重启系统后这些环境变量消失；如需设置永久环境变量，需要将 `export` 语句写入前面所述的 bash 配置文件中)。

内置命令 `readonly` 作用于赋值语句时，和 `declare -r` 类似表示标记变量为只读：

```bash
#如普通只读变量
readonly name[=value]
#如只读索引数组
readonly -a name_index[=(aa bb cc dd ee)]
#如只读关联数组
readonly -A name_associate[=([aa]=1 [bb]=2 [cc]=3 [dd]=4)]
#如标记函数为只读
readonly -f function_name
只读变量不能重新赋值，不能使用内置命令 unset 进行撤销，不能通过命令 declare +r name 取消只读属性。
```

内置命令 read 作用是从标准输入读入一行数据赋值给变量

```bash
$ read NAME
1 2 3           #此处键入字符串 "1 2 3"
$ echo $NAME
1 2 3
```

当有多个变量名时，环境变量 `IFS` 用来将输入分隔成单词。当单词数大于变量数时，剩余的单词和分隔符会被赋值给最后一个变量。当单词数小于变量数时，剩余的变量被赋空值。

```bash
$ read NUM_1 NUM_2 NUM_3
1 2 3 4 5
$ echo $NUM_1
1
$ echo $NUM_2
2
$ echo $NUM_3
3 4 5
```

选项 `-a` 表示将读入的数据赋值给索引数组

```bash
$ read -a BLOG < file #这里输入来自文件，当文件有多行时，第二行及后续行将被忽略。
$ echo ${BLOG[@]}   #取数组中所有元素的值
this is vvpale\'s blog
$ echo ${#BLOG[@]}  #取数组元素个数
4
```

选项 `-p string` 表示在等待输入时显示提示符字符串 string

```bash
$ read -p "请输入变量值：" NUM_4
请输入变量值：345
$ echo $NUM_4
345
```

选项 `-d` 表示指定分隔符 (原分隔符为 \n)

```bash
$ read -d ':' ROOT < /etc/passwd
$ echo $ROOT
root
```

内置命令 `readarray` 和 `mapfile` 表示从标准输入中读入数据并赋值给索引数组，每行赋给一个数组元素：

```bash
$ seq 10 > file
$ readarray NUM <file
$ echo ${NUM[*]}
1 2 3 4 5 6 7 8 9 10
$ echo ${#NUM[*]}
10
```

变量有一个状态 `set/unset`：只要变量被赋过值，就称变量是 `set` 的状态 (即使变量的值为空 `null`)；否则，则称变量是 `unset` 的状态 (即使变量被 `declare` 或其他内置命令声明过)。

可以使用内置命令 `unset` 对变量进行撤销 (特殊变量和只读变量除外)。

```bash
#撤销普通变量
unset name
#撤销整个数组
unset array_name
#撤销数组中单个值 (实际上是把相应的值置空，数组元素个数减一)
unset array_name[index]
#撤销函数
unset function_name
```

对变量进行赋值时，可以使用操作符 `+=` 表示对值的追加：

```bash
#普通变量
$ var=hello
$ echo $var
hello
$ var+=_world
$ echo $var
hello_world
$ unset var
```

```bash
#数字变量使用 += 表示将原有值和新值进行数学运算 (加法)，注意与字符串变量的区别。
$ declare -i NUM=5
$ echo $NUM
5
$ NUM+=5
$ echo $NUM
10
$ unset NUM
$
#数组变量使用 += 作用于上述复合赋值语句表示追加元素至数组
$ array=([0]=hello [1]=world)
$ echo ${array[@]}
hello world
$ echo ${#array[@]}
2
$ array+=(i am vvpale)
$ echo ${array[@]}
hello world i am vvpale
$ echo ${#array[@]}
5
$ unset array
```

## 变量取值/扩展

bash 使用符号 `$` 对变量进行取值，并使用大括号 `{}` 对变量名的起始和结束进行界定，在不引起混淆的情况下，大括号可以省略。

在命令的执行过程中，变量被其值所替换，在替换的过程中能够对应于各种变换。bash 称对变量进行取值的过程为`变量替换`或`变量扩展`。

### 直接取值

```bash
#如果值中包含空白字符，赋值时需要用引号引起来表示一个整体。变量中实际存储的是除去引号的部分。
$ var_1="hello world"
$ echo $var_1
hello world
$
#数组
$ arr=(1000 2000 3000 4000)
$ echo ${arr[@]}
1000 2000 3000 4000
$ echo ${arr[*]}
1000 2000 3000 4000
#注意当被双引号作用时两者的区别 (如前述，同特殊变量 $@和 $* 的情况一致)
```

### 间接引用

在对变量进行取值时，变量名前的符号 `!` 表示对变量的间接引用：

```bash
$ var_2=var_1
$ echo ${!var_2} #必须用大括号
hello world
$
#以上如果写成 ${!var*} 或 ${!var@} 则被替换为所有以 var 为前缀的变量名：
$ echo ${!var*}
var_1 var_2
$ echo ${!var@}
var_1 var_2
#开头的! 如果用在数组变量上则被扩展成数组的所有下标：
$ declare -A array=(["MON"]="星期一" ["TUE"]="星期二" ["WEN"]="星期三" ["THU"]="星期四" ["FRI"]="星期五" ["SAT"]="星期六" ["SUN"]="星期日")
$ echo ${!array[*]}
THU TUE WEN MON FRI SAT SUN
```

### 取长度

在变量名前使用符号 `#` 表示取长度，普通变量表示变量值的字符数，数组变量表示数组参数的个数

```bash
$ echo ${#var_1}
11
$ echo ${#var_2}
5
$ echo ${#array[*]}
7
```

### 判断状态

对于变量 `parameter` 的状态 (`set` 或 `unset`) 和值是否为空(`null`)，bash 提供四种方式扩展：

这里的 `word` 会经过 波浪号扩展 (~ 替换为用户家目录)、变量扩展、命令替换、数学扩展 (以后的文章中会对后两种作详细描述)

`${parameter:-word}` 如果变量状态为 unset 或值为空，返回 `word` 的结果值，否则返回变量的值。

```bash
$ echo ${var_3:-${!var_2}}
hello world
$ var_3=learnning
$ echo ${var_3:-${!var_2}}
learnning
$ echo ${var_4:-~}
/root
```

`${parameter:=word}` 如果变量状态为 unset 或值为空，`word` 的结果会赋值给变量，然后返回变量值。特殊变量 (`$n $$$ $@ $#` 等) 不能用这种方式进行赋值。

```bash
$ echo ${var_4}
                    #变量 var_4 未被赋值，这里输出一个空行
$ echo ${var_4:=${var_3:-${!var_2}}} #注意这里变量 var_3 已被赋值 learnning，所以没有输出 "hello world"
learnning
$ echo ${var_4}                     
learnning
```

`${parameter:?word}` 如果变量状态为 unset 或值为空，`word` 的结果值会被输出到标准错误，如果 shell 是非交互的 (如脚本中) 则退出(exit)；否则展开为变量的值。

```bash
$ echo ${var_4:?"exist"}
learnning
$ echo ${var_5:?"not exist"}
-bash: var_5: not exist
```

`${parameter:+word}` 如果变量状态为 unset 或值为空，什么也不返回，否则返回 `word` 的结果值。

```bash
$ echo ${var_5:+${#var_2}}
                    #变量 var_5 未被赋值，这里输出一个空行
$ echo ${var_4:+${#var_2}}
5
```

以上四种判断变量的方式中，如果省略了冒号 `:`，则表示只判断 `unset` 的情况。

```bash
$ echo ${var_5-"unset"}    
unset
$ var_5=
$ echo ${var_5+"set"}  
set
```

### 取子串

bash 支持使用 `${parameter:offset:length}` 的格式对变量取部分值，其中 `offset` 和 `length` 都是数学表达式，分别代表位置和长度。

`parameter` 为普通变量时，表示从第 offset 个字符 (首字符是第 0 个) 开始，取 length 个字符，如果 `:length` 省略，表示从第 offset 个字符开始，取到变量值的结尾。

```bash
$ echo $var_1
hello world
$ echo ${var_1:6}
world
$ echo ${var_1:1-1:2+3}
hello
```

如果 `offset` 的结果小于 0，则表示从后往前取子串。

```bash
#注意这里为了避免和判断变量状态的写法混淆，冒号和减号之间需要有空白字符或者用括号将负数括起来
$ echo ${var_1: -5}
world
$ echo ${var_1: -5:2}
wo
```

如果 `length` 的结果小于 0，则它表示距离最后一个字符往前 `length` 个字符的位置，和 `offset` 位置一起作用，变量替换的结果就是两个`位置`之间的值。

```bash
$ echo ${var_1:2:-2}
llo wor
```

`parameter` 是 `@` 或使用 `@` 或 `*` 作为下标的数组时，则 `offset` 和 `length` 计算的是元素个数而不是字符数，并且 length 的结果不能为负。

```bash
$ ARRAY=("星期一" "星期二" "星期三" "星期四" "星期五" "星期六" "星期日")
$ echo ${ARRAY[@]:2:3}
星期三 星期四 星期五
$ echo ${ARRAY[@]:(-3)}
星期五 星期六 星期日
#还要注意 $@是从 $1 开始的参数列表和关联数组取结果时的不确定性
$ cat test.sh #将要执行的命令写入脚本
echo $@
echo ${@:0}
echo ${@:2:2}
$
$ ./test.sh 1 2 3 4 5 6 7 8 9 #直接执行脚本，参数列表将赋值给特殊变量 @
1 2 3 4 5 6 7 8 9
./test.sh 1 2 3 4 5 6 7 8 9  #当 offset 为 0 时 (对应脚本第二条命令)，$0 会被添加至参数列表前。
2 3
$ echo ${array[@]} #关联数组输出时，结果和赋值时的元素顺序不一定相同
星期四 星期二 星期三 星期一 星期五 星期六 星期日
$ echo ${array[@]:2:2}
星期二 星期三
```

### 删除

bash 提供两种方式分别删除变量值的前缀或后缀：

`${parameter#word}` 和 `${parameter##word}` 表示删除前缀。`word` 扩展后的结果会作为模式匹配 (通配符匹配，见这里) 变量的值，一个 #表示删除最短匹配前缀，`##` 表示删除最长匹配前缀：

```bash
$ echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/root/bin
$ echo ${PATH#*:}
/usr/local/bin:/usr/sbin:/usr/bin:/root/bin
$ echo ${PATH##*:}
/root/bin
```

`${parameter%word}` 和 `${parameter%%word}` 表示删除后缀。

```bash
$ path=$PWD #赋值语句中等号右边部分也会经过 波浪号扩展、变量扩展、命令替换和数学扩展
$ echo $path
/root/temp/tmp
$ echo ${path%/*} #类似于执行命令 dirname $path
/root/temp
#同样适用于特殊变量和环境变量
$ vim /etc/yum.repos.d/CentOS-Base.repo
$ cd ${_%/*}
$ pwd
/etc/yum.repos.d
$ cd -
/root/temp/tmp
$ echo $BASH_VERSION
4.2.46(1)-release
$ echo ${BASH_VERSION%%[()]*} #注意这里的通配符匹配
4.2.46
```

如果 `parameter` 是 `@` 或 `*` 或以 `@` 或 `*` 作为下标的数组变量，删除操作将作用于每个位置变量或数组的每个参数

```bash
$ echo ${array[@]}
星期四 星期二 星期三 星期一 星期五 星期六 星期日
$ echo ${array[@]#??}
四 二 三 一 五 六 日
```

### 替换

`${parameter/pattern/string}` 的形式表示用 `pattern` 对变量 `parameter` 进行匹配 (通配符匹配)，并使用 `string` 的结果值替换匹配(最长匹配) 的部分。

```bash
$ string=1234567890abcdefghijklmnopqrstuvwxyz
$ echo ${string/1234567890/----}
----abcdefghijklmnopqrstuvwxyz
$ echo ${string/[0-9]/----}
----234567890abcdefghijklmnopqrstuvwxyz
$ echo ${string/a*/....}
1234567890....
```

如果 `pattern` 以字符 `/` 开头，则所有被匹配的结果都被替换

```bash
$ echo ${string//[0-9]/-}
----------abcdefghijklmnopqrstuvwxyz
```

如果 `pattern` 以字符 `#` 开头，匹配的前缀被替换

```bash
$ echo ${string/#*0/---}
---abcdefghijklmnopqrstuvwxyz
```

如果 `pattern` 以字符 `%` 开头，匹配的后缀被替换

```bash
$ echo ${string/%a*/...}
1234567890...
```

使用 `@` 和 `*` 的情况和前述一样，替换将作用于每个参数

```bash
$ A=(100 101 102 103 104) B=.txt P= #多个赋值语句可以写在一行
$ echo ${A[@]}
100 101 102 103 104
$ echo $B
.txt
$ echo -n $P #无输出
$ echo ${A[*]/%$P/$B}
100.txt 101.txt 102.txt 103.txt 104.txt
```

### 大小写转换

`${parameter^pattern}`、`${parameter^^pattern}`、`${parameter,pattern}`、`${parameter,,pattern}`

大小写字母转换，如果 `parameter` 值的首字母匹配模式 `pattern`(通配符匹配，只能是一个字符，可以是 `? * [...]` 或一个英文字母，多个字符不起作用。`pattern` 省略则表示使用 `?`)，则 `^` 将首字母转换成大写，`^^` 将所有匹配字母转换成大写；, 将首字母转换成小写，`,,` 将所有匹配字母转换成小写。

```bash
$ var_5='hello WORLD' var_6='HELLO world'
$ echo ${var_5^[a-z]}
Hello WORLD
$ echo ${var_5^^*}
HELLO WORLD
$ echo ${var_5^^}
HELLO WORLD
$ echo ${var_6,}
hELLO world
$ echo ${var_6,,[A-Z]}
hello world
```

使用 `@` 和 `*` 的情况和前述相同，大小写转换将作用于每个参数

由于 bash 变量赋值的随意性，自定义变量起名时不要和原有变量 (尤其是环境变量) 相冲突，撤销时也要注意不要将环境变量撤销掉(虽然撤销自定义变量并不是必须的)。

```bash
$ unset ${!var*} ${!NUM@} ARRAY ${!arr*} ROOT BLOG NAME path string word A B P
```

## 作用域

bash 变量的作用域分为多种：

1. 写入到 bash 配置文件并用 `export` 导出的环境变量。影响每个启动时加载相应配置文件的 bash 进程及其子进程。
2. 当前 shell 中自定义并通过内置命令 `export` 导出的环境变量。影响当前 bash 进程及其子进程。
3. 当前 shell 中自定义但未导出的变量。影响当前 bash 进程及其子进程 (不包括需要重新初始化 shell 的进程)。
4. 当前 shell 中某个函数中通过内置命令 `local` 自定义的局部变量。只影响此函数及嵌套调用的函数和命令。
5. 当前 shell 中某个命令中的临时变量。只影响此命令。

bash变量作用域涉及到子shell和函数的用法，这里暂时不作举例说明，后续文章中会详细叙述。
