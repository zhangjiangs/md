#### 1、if 相关

##### 1.1、if [ $? -eq 0 \]的含义

if [ $? -eq 0 ]语句代表上一个命令执行后的退出状态

$0: 　　shell或shell脚本的名字
$*:　　 以一对双引号给出参数列表
$@:　 将各个参数分别加双引号返回
$#:    参数的个数
$_:　　代表上一个命令的最后一个参数
$$:　　代表所在命令的PID
$!:　　 代表最后执行的后台命令的PID
$?:　　代表上一个命令执行后的退出状态

```shell
#! /bin/bash
SOME_DIR='/root/cjj/'  

cd $SOME_DIR
if [ $? -eq 0 ]; then  # 检查cd命令是否成功，如果成功才执行rm命令
        rm -rf *txt
else 'Cannot change directory'  # 如果cd命令运行失败，则打印一个错误信息，并退出，返回状态码1
        exit 1
fi
```

##### 1.2、文件/文件夹(目录)判断

```shell
[ -b FILE ] 如果 FILE 存在且是一个块特殊文件则为真。
[ -c FILE ] 如果 FILE 存在且是一个字特殊文件则为真。
[ -d DIR ] 如果 FILE 存在且是一个目录则为真。
[ -e FILE ] 如果 FILE 存在则为真。
[ -f FILE ] 如果 FILE 存在且是一个普通文件则为真。
[ -g FILE ] 如果 FILE 存在且已经设置了SGID则为真。
[ -k FILE ] 如果 FILE 存在且已经设置了粘制位则为真。
[ -p FILE ] 如果 FILE 存在且是一个名字管道(F如果O)则为真。
[ -r FILE ] 如果 FILE 存在且是可读的则为真。
[ -s FILE ] 如果 FILE 存在且大小不为0则为真。
[ -t FD ] 如果文件描述符 FD 打开且指向一个终端则为真。
[ -u FILE ] 如果 FILE 存在且设置了SUID (set user ID)则为真。
[ -w FILE ] 如果 FILE存在且是可写的则为真。
[ -x FILE ] 如果 FILE 存在且是可执行的则为真。
[ -O FILE ] 如果 FILE 存在且属有效用户ID则为真。
[ -G FILE ] 如果 FILE 存在且属有效用户组则为真。
[ -L FILE ] 如果 FILE 存在且是一个符号连接则为真。
[ -N FILE ] 如果 FILE 存在 and has been mod如果ied since it was last read则为真。
[ -S FILE ] 如果 FILE 存在且是一个套接字则为真。
[ FILE1 -nt FILE2 ] 如果 FILE1 has been changed more recently than FILE2, or 如果 FILE1 exists and FILE2 does not则为真。
[ FILE1 -ot FILE2 ] 如果 FILE1 比 FILE2 要老, 或者 FILE2 存在且 FILE1 不存在则为真。
[ FILE1 -ef FILE2 ] 如果 FILE1 和 FILE2 指向相同的设备和节点号则为真。
```

##### 1.3、字符串判断 

```shell
[ -z STRING ] 如果STRING的长度为零则为真 ，即判断是否为空，空即是真；
[ -n STRING ] 如果STRING的长度非零则为真 ，即判断是否为非空，非空即是真；
[ STRING1 = STRING2 ] 如果两个字符串相同则为真 ；
[ STRING1 != STRING2 ] 如果字符串不相同则为真 ；
[ STRING1 ]　 如果字符串不为空则为真,与-n类似
```

##### 1.4、数值判断

```
INT1 -eq INT2           INT1和INT2两数相等为真 ,=
INT1 -ne INT2           INT1和INT2两数不等为真 ,<>
INT1 -gt INT2            INT1大于INT1为真 ,>
INT1 -ge INT2           INT1大于等于INT2为真,>=
INT1 -lt INT2             INT1小于INT2为真 ,<</div>
INT1 -le INT2             INT1小于等于INT2为真,<=
```

##### 1.5、复杂逻辑判断

```
# -a 与 &&
# -o 或 ||
# !  非
```



#### 2、exit0 exit1 的区别

exit（0）：正常运行程序并退出程序；

exit（1）：非正常运行导致退出程序；

exit 0 可以告知你的程序的使用者：你的程序是正常结束的。如果 exit 非 0 值，那么你的程序的使用者通常会认为你的程序产生了一个错误。
在 shell 中调用完你的程序之后，用 echo $? 命令就可以看到你的程序的 exit 值。在 shell 脚本中，通常会根据上一个命令的 $? 值来进行一些流程控制。

#### 3、数组

```
A="a b c def"   # 定义字符串
A=(a b c def)   # 定义字符数组
```

| 命令     | 解释                              | 结果      |
| -------- | --------------------------------- | --------- |
| ${A[@]}  | 返回数组全部元素                  | a b c def |
| ${A[*]}  | 同上                              | a b c def |
| ${A[0]}  | 返回数组第一个元素                | a         |
| ${#A[@]} | 返回数组元素总个数                | 4         |
| ${#A[*]} | 同上                              | 4         |
| ${#A[3]} | 返回第四个元素的长度，即def的长度 | 3         |
| A[3]=xzy | 则是将第四个组数重新定义为 xyz    |           |

#### 4、命令替换

```shell
在bash中，$( )与 ``（反引号）都是用来作命令替换的。
**命令替换**与变量替换差不多，都是用来重组命令行的，先完成引号里的命令行，然后将其结果替换出来，再重组成新的命令行。
```

