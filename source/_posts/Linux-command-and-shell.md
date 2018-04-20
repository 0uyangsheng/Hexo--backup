---
title: 一文了解Linux command and Shell
date: 2018-04-20 15:35:39
tags:
- shell
categories:
- shell
---
{% asset_img bash.png %}

## 系统命令何其多


Linux下命令那么多，好几千个，怎么办？用man查询，如 man ls（查看ls的用法）。

---

看一看有哪些常用命令->

#### 用户管理

- UID，GID

常用命令：`id, who, /etc/passwd, groups`

账号管理： `useradd, passwd, usermod, userdel, groupadd, groupdel, w`

#### 文件管理

常用命令： `pwd, touch, chmod, chown, which, whereis`

- find

查找指定文件并删除：
`find android/ -maxdepth 3 -type f -a -name 2.log -delete`

删除所有文件仅保留特定文件：
`find android/ -type f -not -a -name '*.java' -delete`

查找指定文件并搜索：
`find android/ -type f -a -name '*.java' | grep -rn "activity"`

- 打包： `zip, tar`

例子：Android压缩SDK：
``` shell
tar -zcvf xxx.tar.gz sdk_directory_name --exclude=.repo --exclude=.git --exclude=sdk_directory_name/uboot/build --exclude=sdk_directory_name/out --exclude=sdk_directory_name/ S82_SDK_20141121.tar.gz
```
分卷压缩,网盘上传文件大小有限制这个命令会用到：

`tar -zcvf - .repo/ |split -b 4000M - xxx_sdk.tar.gz`


#### 文件系统

`df, fdisk, mount, lsblk, blkid, /etc/fstab(设置自动挂载)，ln -s(软链接) `

#### 字符处理

grep：`-r : 迭代到子文件夹；  -n：行号；  -i：不区分大小写`

sort：`-n 数字排序   -r 反向排序  -t 指定分隔符`

uniq：删除重复内容

cut：截取文本

#### 网络管理

指定IP地址：`ipconfig eth0 192.168.1.6 netmask 255.255.255.0`

手动打开断开网卡：`ifconfig eth0 up/ifconfig etho down`

查看系统路由表：`route -n`

DNS: `/etc/hosts， /etc/resolv.conf`

#### 进程管理

`ps, top, kill, nice`

#### 正则表达式

[8种字符串截取方法](https://blog.csdn.net/u012359618/article/details/51498959)

- sed与awk的区别：

如果文件是格式化的，即由分隔符分为多个域的，优先使用awk；

awk适合按列（域）操作，sed适合按行操作；

awk适合对文件的抽取整理，sed适合对文件的编辑。



---

## shell编程

#### shell 内建命令

由bash自身提供的命令，而不是/bin下某个可执行文件，比如：`cd，source`。如何确定，通过type。

alias，别名，可以在.bashrc中定制。

任务前后台切换：bg、fg、jobs。可与Ctrl+z、&联合使用。典型场景是运行比较耗时任务。

./， .， source 三者执行shell的区别。

exec：不启动新的shell，而是用要被执行的命令替换当前的shell脚本。exec命令后其他命令将不再执行，且会断开ssh链接。所以一般放到一个子脚本中运行。

source，就是让script在当前shell内执行、 而不是产生一个sub-shell来执行。 由于所有执行结果均在当前shell内执行、而不是产生一个sub-shell来执行。跟 `. xxx.sh` 一样效果。

`./xxx.sh` 是直接fork一个子进程来执行。

export：跨脚本传递变量。

read:从标准输入读取一行。

脚本参数： `$1 第一个参数 $2 第二个参数 …… $@ 所有参数  $# 参数个数  $0 脚本本身 $? 上一条命令返回值`


#### 基础
1. 局部变量，只在某个shell中生效。也可用`local`声明，在函数中生效。
2. 环境变量，也叫全局变量。系统中有预设一些环境变量，如`HOME，PATH`，可以通过 `echo $PATH`访问。
如果需要在shell中导出变量给其他子shell中使用可以通过：`export VAR=value`。
3. 变量的赋值与取值。变量名与值用`=`紧紧相连，中间不能有空格。${}是比$更保险的做法。如果值也是引用变量，要用`""`,如`name="${name1}"`。`unset`可以取消变量。只读变量通过readonly声明，或是`declare -r`。
4. 转义通过`\`来让特殊字符输出。
5. 命令替换是指将标准输出作为值赋给某个变量：`$(命令)`.
6. ()与{}的差别：

    () 将command group置于sub-shell(子shell)中去执行，也称 nested sub-shell。
    {} 则是在同一个shell内完成，也称non-named command group。

7. 常见算术运算符大多需要结合shell的内建命令let来使用。
8. `$(())` 用来作整数运算的.
9. Wildcard与Regular Expression的差别.

    wildcard只作用于argument的path上；而RE却只用于"字符串处理" 的程序中，如某些文字处理工具之间：grep， perl， vi，awk，sed，等等， 常用于表示一段连续的字符串，查找和替换,这与路径名一点关系也没有。



#### 测试判断与循环

测试结构：

`test expression`   or   `[ expression ] 括号内两边有空格` 建议采用后面的方式，更容易跟if  while case 这些连用。

文件测试，常用参数：`-e 文件或目录是否存在；-f 文件是否存在；-d 目录是否存在`。

字符串：`-z 是否为空；-n 非空返回真；= !=`

整数比较：`-eq  -gt  -lt`

逻辑：两种方式 `! -a -o`  or  `! && ||`。

command1 && command2 # command2只有在command1的RV为0(true)的条件下执行。

command1 || command2 # command2 只有在command1的RV为非0(false)的条件下执行。

[]与[[]]的区别[Reference](http://www.zsythink.net/archives/2252)：

当使用‘-n’‘-z’这种判断方式时，‘[]’需要在其中的变量外侧加上双引号，与test命令的用法一致，而使用`[[]]`时不用。

判断某个变量的值是否满足某个正则表达式，可以用符号`=~` + `[[]]`。



if 判断结构：

```
if [ expression ]; then
    cmd1
elif [ exp1 ]; then
    cmd2
else
    cmd3
fi
```

case 判断结构：

```
case VAR in

var1) cmd1;;
var2) cmd2;;
*) cmd3;;

esac
```

for 循环：

```
for VAR in (list)
do

    cmd

done
```

while 循环：

```
while expression
do
    cmd
done
```

until循环结构：（测试假值）。

select循环，是一种菜单扩展循环方式，等待用户输入在执行。

```
select MENU in (list)
do
    cmd
done
```

循环控制：break、continue。


#### 重定向

系统在启动一个进程时会同时打开三个文件：标准输入（stdin）、标准输出（stdout）、标准错误输出（stderr），分别用文件标识符0、1 、2来标识。标准输入为键盘，标准输出及错误输出默认为显示器或是串口。

\> ：标准输出覆盖重定向，会覆盖原始文件。
``` shell

    ls -l /usr/ > ls_usr.txt
    等价于
    ls -l /usr/ 1> ls_usr.txt

```

\>\>: 追加重定向。不清空原始文件。

\>\&: 标识输出重定向，将一个标识的输出重定向到另一个标识的输入。

``` [shell]
    COMMAND > stdout_stderr.txt 2>&1 
    #2>&1代表 错误输出重定向到标准输出，同时打印到文件中。

    2> /dev/null  #丢弃错误输出
```
< : 标准输入重定向。

<<: 这是所谓的here document, 它可以让我们输入一段文本， 直到读到<< 后指定的字符串。比方说：

```
$ cat <<EOF
first line here
second line here
third line here
EOF
```

| ： 管道，将一个命令的输出作为另一个命令的输入。


## 最后 

- 还不错的参考

[shell 十三问](https://github.com/wzb56/13_questions_of_shell)

