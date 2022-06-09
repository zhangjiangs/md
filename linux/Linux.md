# Linux运维管理文档

# Linux用户和用户组管理

## 1、用户和组的关系

用户和用户组的对应关系有以下 4 种：

1. 一对一：一个用户可以存在一个组中，是组中的唯一成员；
2. 一对多：一个用户可以存在多个用户组中，此用户具有这多个组的共同权限；
3. 多对一：多个用户可以存在一个组中，这些用户具有和组相同的权限；
4. 多对多：多个用户可以存在多个组中，也就是以上 3 种关系的扩展。

## 2、UID和GID（用户ID和组ID）

登陆 Linux 系统时，虽然输入的是自己的用户名和密码，但其实 Linux 并不认识你的用户名称，它只认识用户名对应的 ID 号（也就是一串数字）。Linux 系统将所有用户的名称与 ID 的对应关系都存储在 /etc/passwd 文件中。说白了，用户名并无实际作用，仅是为了方便用户的记忆而已。

在 /etc/passwd 文件中，利用 UID 可以找到对应的用户名；在 /etc/group 文件中，利用 GID 可以找到对应的群组名。

## 3、/etc/passwd内容解释

Linux 系统中的 /etc/passwd 文件，是系统用户配置文件，存储了系统中所有用户的基本信息，并且所有用户都可以对此文件执行读操作。

/etc/passwd 文件中的内容非常规律，每行记录对应一个用户。

这些用户中的绝大多数是系统或服务正常运行所必需的用户，这种用户通常称为系统用户或伪用户。系统用户无法用来登录系统，但也不能删除，因为一旦删除，依赖这些用户运行的服务或程序就不能正常执行，会导致系统问题。

不仅如此，每行用户信息都以 "：" 作为分隔符，划分为 7 个字段，每个字段所表示的含义如下：

用户名：密码：UID（用户ID）：GID（组ID）：描述性信息：主目录：默认Shell

### a）用户名

前面讲过，用户名仅是为了方便用户记忆，Linux 系统是通过 UID 来识别用户身份，分配用户权限的。/etc/passwd 文件中就定义了用户名和 UID 之间的对应关系。

### b）密码

"x" 表示此用户设有密码，但不是真正的密码，真正的密码保存在 /etc/shadow 文件中

需要注意的是，虽然 "x" 并不表示真正的密码，但也不能删除，如果删除了 "x"，那么系统会认为这个用户没有密码，从而导致只输入用户名而不用输入密码就可以登陆（只能在使用无密码登录，远程是不可以的），除非特殊情况（如破解用户密码），这当然是不可行的。

### c）UID

UID，也就是用户 ID。每个用户都有唯一的一个 UID，Linux 系统通过 UID 来识别不同的用户。

实际上，UID 就是一个 0~65535 之间的数，不同范围的数字表示不同的用户身份，具体如下表所示。

| UID 范围  | 用户身份                                                     |
| --------- | ------------------------------------------------------------ |
| 0         | 超级用户。UID 为 0 就代表这个账号是管理员账号。在 Linux 中，如何把普通用户升级成管理员呢？只需把其他用户的 UID 修改为 0 就可以了，这一点和 Windows 是不同的。不过不建议建立多个管理员账号。 |
| 1~499     | 系统用户（伪用户）。也就是说，此范围的 UID 保留给系统使用。其中，1~99 用于系统自行创建的账号；100~499 分配给有系统账号需求的用户。  其实，除了 0 之外，其他的 UID 并无不同，这里只是默认 500 以下的数字给系统作为保留账户，只是一个公认的习惯而已。 |
| 500~65535 | 普通用户。通常这些 UID 已经足够用户使用了。但不够用也没关系，2.6.x 内核之后的 Linux 系统已经可以支持 232 个 UID 了。 |

### d）GID

全称“Group ID”，简称“组ID”，表示用户初始组的组 ID 号。这里需要解释一下初始组和附加组的概念。

初始组，指用户登陆时就拥有这个用户组的相关权限。每个用户的初始组只能有一个，通常就是将和此用户的用户名相同的组名作为该用户的初始组。比如说，我们手工添加用户 lamp，在建立用户 lamp 的同时，就会建立 lamp 组作为 lamp 用户的初始组。

附加组，指用户可以加入多个其他的用户组，并拥有这些组的权限。每个用户只能有一个初始组，除初始组外，用户再加入其他的用户组，这些用户组就是这个用户的附加组。附加组可以有多个，而且用户可以有这些附加组的权限。

举例来说，刚刚的 lamp 用户除属于初始组 lamp 外，我又把它加入了 users 组，那么 lamp 用户同时属于 lamp 组和 users 组，其中 lamp 是初始组，users 是附加组。

当然，初始组和附加组的身份是可以修改的，但是我们在工作中不修改初始组，只修改附加组，因为修改了初始组有时会让管理员逻辑混乱。

需要注意的是，在 /etc/passwd 文件的第四个字段中看到的 ID 是这个用户的初始组。

### e）描述性信息

这个字段并没有什么重要的用途，只是用来解释这个用户的意义而已。

### f）主目录

也就是用户登录后有操作权限的访问目录，通常称为用户的主目录、家目录。

### g）默认的shell

Shell 就是 Linux 的命令解释器，是用户和 Linux 内核之间沟通的桥梁。

我们知道，用户登陆 Linux 系统后，通过使用 Linux 命令完成操作任务，但系统只认识类似 0101 的机器语言，这里就需要使用命令解释器。也就是说，Shell 命令解释器的功能就是将用户输入的命令转换成系统可以识别的机器语言。

通常情况下，Linux 系统默认使用的命令解释器是 bash（/bin/bash），当然还有其他命令解释器，例如 sh、csh 等。

在 /etc/passwd 文件中，大家可以把这个字段理解为用户登录之后所拥有的权限。如果这里使用的是 bash 命令解释器，就代表这个用户拥有权限范围内的所有权限。例如：

```
[root@localhost ~]# vi /etc/passwd
lamp:x:502:502::/home/lamp:/bin/bash
```

我手工添加了 lamp 用户，它使用的是 bash 命令解释器，那么这个用户就可以使用普通用户的所有权限。

如果我把 lamp 用户的 Shell 命令解释器修改为 /sbin/nologin，那么，这个用户就不能登录了，例如：

```
[root@localhost ~]# vi /etc/passwd
lamp:x:502:502::/home/lamp:/sbin/nologin
```

因为 /sbin/nologin 就是禁止登录的 Shell。同样，如果我在这里放入的系统命令，如 /usr/bin/passwd，例如：

```
[root@localhost ~]#vi /etc/passwd
lamp:x:502:502::/home/lamp:/usr/bin/passwd
```

那么这个用户可以登录，但登录之后就只能修改自己的密码。但是，这里不能随便写入和登陆没有关系的命令（如 ls），系统不会识别这些命令，同时也就意味着这个用户不能登录。

## 4、/etc/shadow（影子文件）内容解析

/etc/shadow 文件，用于存储 Linux 系统中用户的密码信息，又称为“影子文件”。

前面介绍了 /etc/passwd 文件，由于该文件允许所有用户读取，易导致用户密码泄露，因此 Linux 系统将用户的密码信息从 /etc/passwd 文件中分离出来，并单独放到了此文件中。

/etc/shadow 文件只有 root 用户拥有读权限，其他用户没有任何权限，这样就保证了用户密码的安全性。

<!--注意，如果这个文件的权限发生了改变，则需要注意是否是恶意攻击。-->

看看shadow文件的内容：

```
[root@Test-BK-Dev /]# cat /etc/shadow
root:$6$2PqOIU6S$xfieTLMkk0KD2DcamJKQ1FWNzVR9ZqyRKle5M7TgaBV.Fx0NSlnmLgksR5IwBQjagNeO2iEwB.JzEKaIY3S/r1:18800:0:99999:7:::
bin:*:17834:0:99999:7:::
daemon:*:17834:0:99999:7:::
adm:*:17834:0:99999:7:::
............省略部分内容
zhangjs:!!:18816:0:99999:7:::
test:!!:18816:0:99999:7:::
```

同 /etc/passwd 文件一样，文件中每行代表一个用户，同样使用 ":" 作为分隔符，不同之处在于，每行用户信息被划分为 9 个字段。每个字段的含义如下：

用户名：加密密码：最后一次修改时间：最小修改时间间隔：密码有效期：密码需要变更前的警告天数：密码过期后的宽限时间：账号失效时间：保留字段

接下来解释9个字段的意义：

### a）用户名

同 /etc/passwd 文件的用户名有相同的含义。

### b）加密密码

这里保存的是真正加密的密码。目前 Linux 的密码采用的是 SHA512 散列加密算法，原来采用的是 MD5 或 DES 加密算法。SHA512 散列加密算法的加密等级更高，也更加安全。

<!--注意，这串密码产生的乱码不能手工修改，如果手工修改，系统将无法识别密码，导致密码失效。很多软件透过这个功能，在密码串前加上 "!"、"*" 或 "x" 使密码暂时失效。-->

所有伪用户的密码都是 "!!" 或 "*"，代表没有密码是不能登录的。当然，新创建的用户如果不设定密码，那么它的密码项也是 "!!"，代表这个用户没有密码，不能登录。

### c）最后一次修改时间

此字段表示最后一次修改密码的时间

root账号的时间显示是18800，Linux 计算日期的时间是以 1970 年 1 月 1 日作为 1 不断累加得到的时间，到 1971 年 1 月 1 日，则为 366 天。这里显示 18800 天，也就是说，此 root 账号在 1970 年 1 月 1 日之后的第 18800天修改的 root 用户密码。通过如下方式换算时间：

```shell
[root@Test-BK-Dev /]# date -d "1970-01-01 18800 days"
Tue Jun 22 00:00:00 CST 2021
```

### d）最小修改时间间隔

最小修改间隔时间，也就是说，该字段规定了从第 3 字段（最后一次修改密码的日期）起，多长时间之内不能修改密码。如果是 0，则密码可以随时修改；如果是 10，则代表密码修改后 10 天之内不能再次修改密码。

此字段是为了针对某些人频繁更改账户密码而设计的。

### e）密码有效期

经常变更密码是个好习惯，为了强制要求用户变更密码，这个字段可以指定距离第 3 字段（最后一次更改密码）多长时间内需要再次变更密码，否则该账户密码进行过期阶段。
该字段的默认值为 99999，也就是 273 年，可认为是永久生效。如果改为 90，则表示密码被修改 90 天之后必须再次修改，否则该用户即将过期。管理服务器时，通过这个字段强制用户定期修改密码。

### f）密码需要变更前的警告天数

与第 5 字段相比较，当账户密码有效期快到时，系统会发出警告信息给此账户，提醒用户 "再过 n 天你的密码就要过期了，请尽快重新设置你的密码！"。

该字段的默认值是 7，也就是说，距离密码有效期的第 7 天开始，每次登录系统都会向该账户发出 "修改密码" 的警告信息。

### g）密码过期后的宽限天数

也称为“口令失效日”，简单理解就是，在密码过期后，用户如果还是没有修改密码，则在此字段规定的宽限天数内，用户还是可以登录系统的；如果过了宽限天数，系统将不再让此账户登陆，也不会提示账户过期，是完全禁用。

比如说，此字段规定的宽限天数是 10，则代表密码过期 10 天后失效；如果是 0，则代表密码过期后立即失效；如果是 -1，则代表密码永远不会失效。

### h）账号失效时间

同第 3 个字段一样，使用自 1970 年 1 月 1 日以来的总天数作为账户的失效时间。该字段表示，账号在此字段规定的时间之外，不论你的密码是否过期，都将无法使用！

该字段通常被使用在具有收费服务的系统中。

### i）保留

这个字段目前没有使用，等待新功能的加入。

### 忘记密码怎么办

对于普通账户的密码遗失，可以通过 root 账户解决，它会重新给你配置好指定账户的密码，而不需知道你原有的密码（利用 root 的身份使用 passwd 命令即可）。

如果 root 账号的密码遗失，则需要重新启动进入单用户模式，系统会提供 root 权限的 bash 接口，此时可以用 passwd 命令修改账户密码；也可以通过挂载根目录，修改 /etc/shadow，将账户的 root 密码清空的方法，此方式可使用 root 无法密码即可登陆，建议登陆后使用 passwd 命令配置 root 密码。

## 5、/etc/group文件解析

/ect/group 文件是用户组配置文件，即用户组的所有信息都存放在此文件中。

此文件是记录组 ID（GID）和组名相对应的文件。前面讲过，etc/passwd 文件中每行用户信息的第四个字段记录的是用户的初始组 ID，那么，此 GID 的组名到底是什么呢？就要从 /etc/group 文件中查找。

```shell
[root@Test-BK-Dev ~]# cat /etc/group
root:x:0:
bin:x:1:
daemon:x:2:
...........内容省略
zabbix:x:994:
ntp:x:38:
zhangjs:x:1002:
test:x:1003:
```

此文件中每一行各代表一个用户组，我们曾创建 test用户，系统默认生成一个test用户组，在此可以看到，此用户组的 GID 为 1003，目前它仅作为 test用户的初始组。

各用户组中，还是以 "：" 作为字段之间的分隔符，分为 4 个字段，每个字段对应的含义为：

组名：密码：GID：该用户组中的用户列表

接下来，分别介绍各个字段具体的含义。

### a）组名

也就是是用户组的名称，有字母或数字构成。同 /etc/passwd 中的用户名一样，组名也不能重复。

### b）组密码

和 /etc/passwd 文件一样，这里的 "x" 仅仅是密码标识，真正加密后的组密码默认保存在 /etc/gshadow 文件中。

不过，用户设置密码是为了验证用户的身份，那用户组设置密码是用来做什么的呢？用户组密码主要是用来指定组管理员的，由于系统中的账号可能会非常多，root 用户可能没有时间进行用户的组调整，这时可以给用户组指定组管理员，如果有用户需要加入或退出某用户组，可以由该组的组管理员替代 root 进行管理。但是这项功能目前很少使用，我们也很少设置组密码。如果需要赋予某用户调整某个用户组的权限，则可以使用 sudo 命令代替。

### c）组ID (GID)

就是群组的 ID 号，Linux 系统就是通过 GID 来区分用户组的，同用户名一样，组名也只是为了便于管理员记忆。

这里的组 GID 与 /etc/passwd 文件中第 4 个字段的 GID 相对应，实际上，/etc/passwd 文件中使用 GID 对应的群组名，就是通过此文件对应得到的。

### d）组中的用户

此字段列出每个群组包含的所有用户。需要注意的是，如果该用户组是这个用户的初始组，则该用户不会写入这个字段，可以这么理解，该字段显示的用户都是这个用户组的附加用户。

举个例子，test组的组信息为

```
 test:x:1003:
```

可以看到，第四个字段没有写入 test用户，因为 test 组是test用户的初始组。如果要查询这些用户的初始组，则需要先到 /etc/passwd 文件中查看 GID（第四个字段），然后到 /etc/group 文件中比对组名。

每个用户都可以加入多个附加组，但是只能属于一个初始组。所以我们在实际工作中，如果需要把用户加入其他组，则需要以附加组的形式添加。例如，我们想让test也加入 root 这个群组，那么只需要在第一行的最后一个字段加入test，即 

```shell
root:x:0:test
```

就可以了。一般情况下，用户的初始组就是在建立用户的同时建立的和用户名相同的组。

到此，我们已经学习了/etc/passwd、/etc/shadow、/etc/group，它们之间的关系可以这样理解，即先在 /etc/group 文件中查询用户组的 GID 和组名；然后在 /etc/passwd 文件中查找该 GID 是哪个用户的初始组，同时提取这个用户的用户名和 UID；最后通过 UID 到 /etc/shadow 文件中提取和这个用户相匹配的密码。

## 6、/etc/gshadow文件内容解析

先看下gshadow文件的内容：

```shell
[root@Test-BK-Dev /]# cat /etc/gshadow
root:::
bin:::
daemon:::
sys:::
................内容省略
smmsp:!::
zabbix:!::
ntp:!::
zhangjs:!::
test:!::
```

文件中，每行代表一个组用户的密码信息，各行信息用 ":" 作为分隔符分为 4 个字段，每个字段的含义如下：

组名：加密密码：组管理员：组附加用户列表

### a）组名

同 /etc/group 文件中的组名相对应。

### b）组密码

对于大多数用户来说，通常不设置组密码，因此该字段常为空，但有时为 "!"，指的是该群组没有组密码，也不设有群组管理员。

### c）组管理员

从系统管理员的角度来说，该文件最大的功能就是创建群组管理员。那么，什么是群组管理员呢？

考虑到 Linux 系统中账号太多，而超级管理员 root 可能比较忙碌，因此当有用户想要加入某群组时，root 或许不能及时作出回应。这种情况下，如果有群组管理员，那么他就能将用户加入自己管理的群组中，也就免去麻烦 root 了。

不过，由于目前有 sudo 之类的工具，因此群组管理员的这个功能已经很少使用了。

### d）组中的附加用户

该字段显示这个用户组中有哪些附加用户，和 /etc/group 文件中附加组显示内容相同。

## 7、初始组和附加组

一个用户可以所属多个群组，并同时拥有这些群组的权限，这就引出了初始组（有时也称主组）和附加组。

```shell
[root@Test-BK-Dev /]# usermod -G zhangjs test
[root@Test-BK-Dev /]#  grep "test" /etc/passwd /etc/group /etc/gshadow
/etc/passwd:test:x:1002:1003::/home/test:/bin/bash
/etc/group:zhangjs:x:1002:test
/etc/group:test:x:1003:
/etc/gshadow:zhangjs:!::test
/etc/gshadow:test:!::
```

可以看出test用户已经被加入到zhangjs组的附加组里

一个用户可以所属多个附加组，但只能有一个初始组。那么，如何知道某用户所属哪些群组呢？使用 groups 命令即可。

```shell
[root@Test-BK-Dev /]# groups test
test : test zhangjs
```

通过以上输出信息可以得知，test用户同时属于 test群组和zhangjs群组，而且，第一个出现的为用户的初始组，后面的都是附加组，所以test用户的初始组为 test 群组，附加组为 zhangjs 群组。

## 8、/etc/login.defs：创建用户的默认设置文件

/etc/login.defs 文件用于在创建用户时，对用户的一些基本属性做默认设置，例如指定用户 UID 和 GID 的范围，用户的过期时间，密码的最大长度，等等。

需要注意的是，该文件的用户默认配置对 root 用户无效。并且，当此文件中的配置与 /etc/passwd 和 /etc/shadow 文件中的用户信息有冲突时，系统会以/etc/passwd 和 /etc/shadow 为准。

​                                                                       表/etc/login.defs文件内容

| 设置项                   | 含义                                                         |
| ------------------------ | ------------------------------------------------------------ |
| MAIL_DIR /var/spool/mail | 创建用户时，系统会在目录 /var/spool/mail 中创建一个用户邮箱，比如 lamp 用户的邮箱是 /var/spool/mail/lamp。 |
| PASS_MAX_DAYS 99999      | 密码有效期，99999 是自 1970 年 1 月 1 日起密码有效的天数，相当于 273 年，可理解为密码始终有效。 |
| PASS_MIN_DAYS 0          | 表示自上次修改密码以来，最少隔多少天后用户才能再次修改密码，默认值是 0。 |
| PASS_MIN_LEN 5           | 指定密码的最小长度，默认不小于 5 位，但是现在用户登录时验证已经被 PAM 模块取代，所以这个选项并不生效。 |
| PASS_WARN_AGE 7          | 指定在密码到期前多少天，系统就开始通过用户密码即将到期，默认为 7 天。 |
| UID_MIN 500              | 指定最小 UID 为 500，也就是说，添加用户时，默认 UID 从 500 开始。注意，如果手工指定了一个用户的 UID 是 550，那么下一个创建的用户的 UID 就会从 551 开始，哪怕 500~549 之间的 UID 没有使用。 |
| UID_MAX 60000            | 指定用户最大的 UID 为 60000。                                |
| GID_MIN 500              | 指定最小 GID 为 500，也就是在添加组时，组的 GID 从 500 开始。 |
| GID_MAX 60000            | 用户 GID 最大为 60000。                                      |
| CREATE_HOME yes          | 指定在创建用户时，是否同时创建用户主目录，yes 表示创建，no 则不创建，默认是 yes。 |
| UMASK 077                | 用户主目录的权限默认设置为 077。                             |
| USERGROUPS_ENAB yes      | 指定删除用户的时候是否同时删除用户组，准备地说，这里指的是删除用户的初始组，此项的默认值为 yes。 |
| ENCRYPT_METHOD SHA512    | 指定用户密码采用的加密规则，默认采用 SHA512，这是新的密码加密模式，原先的 Linux 只能用 DES 或 MD5 加密。 |

## 9、useradd命令详解：添加新的系统用户

Linux 系统中，可以使用 useradd 命令新建用户，此命令的基本格式如下：

```shell
[root@localhost ~]#useradd [选项] 用户名
```

该命令常用的选项及各自的含义，如表 1 所示。

| 选项                    | 含义                                                         |
| ----------------------- | ------------------------------------------------------------ |
| -u     UID              | 手工指定用户的 UID，注意 UID 的范围（不要小于 500）。        |
| -d   主目录             | 手工指定用户的主目录。主目录必须写绝对路径，而且如果需要手工指定主目录，则一定要注意权限； |
| -c             用户说明 | 手工指定/etc/passwd文件中各用户信息中第 5 个字段的描述性内容，可随意配置； |
| -g           组名       | 手工指定用户的初始组。一般以和用户名相同的组作为用户的初始组，在创建用户时会默认建立初始组。一旦手动指定，则系统将不会在创建此默认的初始组目录。 |
| -G 组名                 | 指定用户的附加组。我们把用户加入其他组，一般都使用附加组；   |
| -s shell                | 手工指定用户的登录 Shell，默认是 /bin/bash；                 |
| -e 曰期                 | 指定用户的失效曰期，格式为 "YYYY-MM-DD"。也就是 /etc/shadow 文件的第八个字段； |
| -o                      | 允许创建的用户的 UID 相同。例如，执行 "useradd -u 0 -o usertest" 命令建立用户 usertest，它的 UID 和 root 用户的 UID 相同，都是 0； |
| -m                      | 建立用户时强制建立用户的家目录。在建立系统用户时，该选项是默认的； |
| -r                      | 创建系统用户，也就是 UID 在 1~499 之间，供系统程序使用的用户。由于系统用户主要用于运行系统所需服务的权限配置，因此系统用户的创建默认不会创建主目录。 |

useradd 命令创建用户的过程，其实就是修改了与用户相关的几个文件或目录，前面章节已经对这些文件做了详细介绍。

其实，系统已经帮我们规定了非常多的默认值，在没有特殊要求下，无需使用任何选项即可成功创建用户。例如：

```shell
[root@localhost ~]# useradd lamp
```

此行命令就表示创建 lamp 普通用户。加参数后可以自定义创建用户的属性

useradd 命令在添加用户时参考的默认值文件主要有两个，分别是 /etc/default/useradd 和 /etc/login.defs

/etc/login.defs文件在前一章节已经做了介绍，接下来看/etc/default/useradd文件

## 10、/etc/default/useradd 文件

可以直接通过命令进行查看，结果是一样的：

```shell
[root@Test-BK-Dev mail]# useradd -D
GROUP=100
HOME=/home
INACTIVE=-1
EXPIRE=
SHELL=/bin/bash
SKEL=/etc/skel
CREATE_MAIL_SPOOL=yes
```

-D 选项指的就是查看新建用户的默认值。

也可以通过查看文件的方式

```shell
[root@Test-BK-Dev mail]# cat /etc/default/useradd
# useradd defaults file
GROUP=100
HOME=/home
INACTIVE=-1
EXPIRE=
SHELL=/bin/bash
SKEL=/etc/skel
CREATE_MAIL_SPOOL=yes

```

| 参数                  | 含义                                                         |
| --------------------- | ------------------------------------------------------------ |
| GR0UP=100             | 这个选项用于建立用户的默认组，也就是说，在添加每个用户时，用户的初始组就是 GID 为 100 的这个用户组。但 CentOS 并不是这样的，而是在添加用户时会自动建立和用户名相同的组作为此用户的初始组。也就是说这个选项并不会生效。  Linux 中默认用户组有两种机制：一种是私有用户组机制，系统会创建一个和用户名相同的用户组作为用户的初始组；另一种是公共用户组机制，系统用 GID 是 100 的用户组作为所有新建用户的初始组。目前我们采用的是私有用户组机制。 |
| HOME=/home            | 指的是用户主目录的默认位置，所有新建用户的主目录默认都在 /home/下，刚刚新建的 lamp1 用户的主目录就为 /home/lamp1/。 |
| INACTIVE=-1           | 指的是密码过期后的宽限天数，也就是 /etc/shadow 文件的第七个字段。这里默认值是 -1，代表所有新建立的用户密码永远不会失效。 |
| EXPIRE=               | 表示密码失效时间，也就是 /etc/shadow 文件的第八个字段。默认值是空，代表所有新建用户没有失效时间，永久有效。 |
| SHELL=/bin/bash       | 表示所有新建立的用户默认 Shell 都是 /bin/bash。              |
| SKEL=/etc/skel        | 在创建一个新用户后，你会发现，该用户主目录并不是空目录，而是有 .bash_profile、.bashrc 等文件，这些文件都是从 /etc/skel 目录中自动复制过来的。因此，更改 /etc/skel 目录下的内容就可以改变新建用户默认主目录中的配置文件信息。 |
| CREATE_MAIL_SPOOL=yes | 指的是给新建用户建立邮箱，默认是创建。也就是说，对于所有的新建用户，系统都会新建一个邮箱，放在 /var/spool/mail/ 目录下，和用户名相同。例如，lamp1 的邮箱位于 /var/spool/mail/lamp1。 |

注意，此文件中各选项值的修改方式有 2 种，一种是通过 Vim 文本编辑器手动修改，另一种就是使用文章开头介绍的 useradd 命令，不过所用的命令格式发生了改变：

```shell
useradd -D [选项] 参数
```

用此命令修改 /etc/default/useradd 文件，可使用的选项如下表所示。

| 选项+参数   | 含义                                                         |
| ----------- | ------------------------------------------------------------ |
| -b HOME     | 设置所创建的主目录所在的默认目录，只需用目录名替换 HOME 即可，例如 useradd -D -b /gargae。 |
| -e EXPIRE   | 设置密码失效时间，EXPIRE 参数应使用 YYYY-MM-DD 格式，例如 useradd -D -e 2019-10-17。 |
| -f INACTIVE | 设置密码过期的宽限天数，例如 useradd -D -f 7。               |
| -g GROUP    | 设置新用户所在的初始组，例如 useradd -D -g bear。            |
| -s SHELL    | 设置新用户的默认 shell，SHELL 必须是完整路径，例如 useradd -D -s /usr/bin/csh。 |

通过 /etc/default/useradd 文件，大家仅能修改有关新用户的部分默认值，有一些内容并没有在这个文件中，例如修改用户默认的 UID、GID，以及对用户密码的默认设置，对这些默认值的修改就需要在 /etc/login.defs 文件中进行。

其实，useradd 命令创建用户的过程是这样的，系统首先读取 /etc/login.defs 和 /etc/default/useradd，根据这两个配置文件中定义的规则添加用户，也就是向 /etc/passwd、/etc/group、/etc/shadow、/etc/gshadow 文件中添加用户数据，接着系统会自动在 /etc/default/useradd 文件设定的目录下建立用户主目录，最后复制 /etc/skel 目录中的所有文件到此主目录中，由此，一个新的用户就创建完成了。

当然，如果你能彻底掌握 useradd 命令创建用户的整个过程，完全可以手动创建用户。

## 11、passwd命令：修改用户密码

passwd 命令的基本格式如下：

```shell
[root@localhost ~]#passwd [选项] 用户名
```

选项：

- -S：查询用户密码的状态，也就是 /etc/shadow 文件中此用户密码的内容。仅 root 用户可用；
- -l：暂时锁定用户，该选项会在 /etc/shadow 文件中指定用户的加密密码串前添加 "!"，使密码失效。仅 root 用户可用；
- -u：解锁用户，和 -l 选项相对应，也是只能 root 用户使用；
- --stdin：可以将通过管道符输出的数据作为用户的密码。主要在批量添加用户时使用；
- -n 天数：设置该用户修改密码后，多长时间不能再次修改密码，也就是修改 /etc/shadow 文件中各行密码的第 4 个字段；
- -x 天数：设置该用户的密码有效期，对应 /etc/shadow 文件中各行密码的第 5 个字段；
- -w 天数：设置用户密码过期前的警告天数，对于 /etc/shadow 文件中各行密码的第 6 个字段；
- -i 日期：设置用户密码失效日期，对应 /etc/shadow 文件中各行密码的第 7 个字段。

不加用户名的话就是修改当前用户的密码

可以看到，与使用 root 账户修改普通用户的密码不同，普通用户修改自己的密码需要先输入自己的旧密码，只有旧密码输入正确才能输入新密码。不仅如此，此种修改方式对密码的复杂度有严格的要求，新密码太短、太简单，都会被系统检测出来并禁止用户使用。

```
很多Linux 发行版为了系统安装，都使用了 PAM 模块进行密码的检验，设置密码太短、与用户名相同、是常见字符串等，都会被 PAM 模块检查出来，从而禁止用户使用此类密码。有关 PAM 模块，后续章节会进行详细介绍。
```

而使用 root 用户，无论是修改普通用户的密码，还是修改自己的密码，都可以不遵守 PAM 模块设定的规则，就比如我刚刚给 lamp 用户设定的密码是 "123"，系统虽然会提示密码过短和过于简单，但依然可以设置成功。

如果需要修改账号信息，如密码有效期等，也可以直接修改/etc/shadow文件

Linux锁定账号实际上就是在shadow文件中的加密密码之前加了“！！”让密码失效而已，如下：

```shell
[root@Test-BK-Dev ~]# cat /etc/shadow
root:$6$2PqOIU6S$xfieTLMkk0KD2DcamJKQ1FWNzVR9ZqyRKle5M7TgaBV.Fx0NSlnmLgksR5IwBQjagNeO2iEwB.JzEKaIY3S/r1:18800:0:99999:7:::
bin:*:17834:0:99999:7:::
…………………………省略内容
ntp:!!:18605::::::
zhangjs:!!:18816:0:99999:7:::
test:$6$WGVZAuyA$BLjmG4UHRef.ZelJksNwwkDGMI8p4QsOB9wPr8vOkE9ECvwvT88dT2L5HtszB7tDgyS7EirF5/B7ZYHvMgSJm/:18817:0:99999:7:::
[root@Test-BK-Dev ~]# passwd -l test
Locking password for user test.
passwd: Success
[root@Test-BK-Dev ~]# cat /etc/shadow
root:$6$2PqOIU6S$xfieTLMkk0KD2DcamJKQ1FWNzVR9ZqyRKle5M7TgaBV.Fx0NSlnmLgksR5IwBQjagNeO2iEwB.JzEKaIY3S/r1:18800:0:99999:7:::
bin:*:17834:0:99999:7:::
daemon:*:17834:0:99999:7:::
……………………省略内容
ntp:!!:18605::::::
zhangjs:!!:18816:0:99999:7:::
test:!!$6$WGVZAuyA$BLjmG4UHRef.ZelJksNwwkDGMI8p4QsOB9wPr8vOkE9ECvwvT88dT2L5HtszB7tDgyS7EirF5/B7ZYHvMgSJm/:18817:0:99999:7:::

```

```shell
#调用管道符，给 test 用户设置密码 "123456"
[root@Test-BK-Dev ~]# echo "123456" | passwd --stdin test
Changing password for user test.
passwd: all authentication tokens updated successfully.
```

为了方便系统管理，passwd 命令提供了 --stdin 选项，用于批量给用户设置初始密码。

使用此方式批量给用户设置初始密码，当然好处就是方便快捷，但需要注意的是，这样设定的密码会把密码明文保存在历史命令中，如果系统被攻破，别人可以在 /root/.bash_history 中找到设置密码的这个命令，存在安全隐患。

因此，如果使用这种方式修改密码，那么应该记住两件事情：第一，手工清除历史命令；第二，强制这些新添加的用户在第一次登录时必须修改密码（具体方法参考 "chage" 命令）。

注意，并非所有 Linux 发行版都支持使用此选项，使用之前可以使用 man passwd 命令确认当前系统是否支持。

## 12、usermod命令：修改用户信息

当我们需要修改用户信息的时候有两个方法：

一个方法是手动修改涉及用户信息的相关文件（/etc/passwd、/etc/shadow、/etc/group、/etc/gshadow），另一个方法就是 usermod 命令，该命令专门用于修改用户信息。

usermod 命令的基本格式如下：

```shell
[root@localhost ~]#usermod [选项] 用户名
```

选项：

- -c 用户说明：修改用户的说明信息，即修改 /etc/passwd 文件目标用户信息的第 5 个字段；
- -d 主目录：修改用户的主目录，即修改 /etc/passwd 文件中目标用户信息的第 6 个字段，需要注意的是，主目录必须写绝对路径；
- -e 日期：修改用户的失效曰期，格式为 "YYYY-MM-DD"，即修改 /etc/shadow 文件目标用户密码信息的第 8 个字段；
- -g 组名：修改用户的初始组，即修改 /etc/passwd 文件目标用户信息的第 4 个字段（GID）；
- -u UID：修改用户的UID，即修改 /etc/passwd 文件目标用户信息的第 3 个字段（UID）；
- -G 组名：修改用户的附加组，其实就是把用户加入其他用户组，即修改 /etc/group 文件；
- -l 用户名：修改用户名称；
- -L：临时锁定用户（Lock）；
- -U：解锁用户（Unlock），和 -L 对应；
- -s shell：修改用户的登录 Shell，默认是 /bin/bash。

如果你仔细观察会发现，其实 usermod 命令提供的选项和 useradd 命令的选项相似，因为 usermod 命令就是用来调整使用 useradd 命令添加的用户信息的。

不过，相比 useradd 命令，usermod 命令还多出了几个选项，即 -L 和 -U，作用分别与 passwd 命令的 -l 和-u 相同。需要注意的是，并不是所有的 Linux 发行版都包含这个命令，因此，使用前可以使用 man usermod 命令确定系统是否支持。

<!--此命令对用户的临时锁定，同 passwd 命令一样，都是在 /etc/passwd 文件目标用户的加密密码字段前添加 "!"，使密码失效；反之，解锁用户就是将添加的 "!" 去掉。-->

## 13、chage用法详解：修改用户密码状态

除了 `passwd -S` 命令可以查看用户的密码信息外，还可以利用 chage 命令，它可以显示更加详细的用户密码信息，并且和 passwd 命令一样，提供了修改用户密码信息的功能。

<!--如果你要修改用户的密码信息，我个人建议，还是直接修改 /etc/shadow 文件更加方便。-->

首先，我们来看 chage 命令的基本格式：

```shell
[root@localhost ~]#chage [选项] 用户名
```

选项：

- -l：列出用户的详细密码状态;
- -d 日期：修改 /etc/shadow 文件中指定用户密码信息的第 3 个字段，也就是最后一次修改密码的日期，格式为 YYYY-MM-DD；
- -m 天数：修改密码最短保留的天数，也就是 /etc/shadow 文件中的第 4 个字段；
- -M 天数：修改密码的有效期，也就是 /etc/shadow 文件中的第 5 个字段；
- -W 天数：修改密码到期前的警告天数，也就是 /etc/shadow 文件中的第 6 个字段；
- -i 天数：修改密码过期后的宽限天数，也就是 /etc/shadow 文件中的第 7 个字段；
- -E 日期：修改账号失效日期，格式为 YYYY-MM-DD，也就是 /etc/shadow 文件中的第 8 个字段。

 chage 命令除了修改密码信息的功能外，还可以强制用户在第一次登录后，必须先修改密码，并利用新密码重新登陆系统，此用户才能正常使用。

```shell
[root@localhost ~]#chage -d 0 test
```

## 14、userdel命令详解：删除用户

userdel 命令功能很简单，就是删除用户的相关数据。此命令只有 root 用户才能使用。

通过前面的学习我们知道，用户的相关数据包含如下几项：

- 用户基本信息：存储在 /etc/passwd 文件中；
- 用户密码信息：存储在 /etc/shadow 文件中；
- 用户群组基本信息：存储在 /etc/group 文件中；
- 用户群组信息信息：存储在 /etc/gshadow 文件中；
- 用户个人文件：主目录默认位于 /home/用户名，邮箱位于 /var/spool/mail/用户名。

其实，userdel 命令的作用就是从以上文件中，删除与指定用户有关的数据信息。


userdel 命令的语法很简单，基本格式如下：

```shell
[root@localhost ~]# userdel -r 用户名
```

-r 选项表示在删除用户的同时删除用户的家目录。

<!--注意，在删除用户的同时如果不删除用户的家目录，那么家目录就会变成没有属主和属组的目录，也就是垃圾文件。-->

<!--最后需要大家注意的是，如果要删除的用户已经使用过系统一段时间，那么此用户可能在系统中留有其他文件，因此，如果我们想要从系统中彻底的删除某个用户，最好在使用 userdel 命令之前，先通过 `find -user 用户名` 命令查出系统中属于该用户的文件，然后在加以删除。-->

## 15、id命令：查看用户的UID和GID

id 命令可以查询用户的UID、GID 和附加组的信息。命令比较简单，格式如下：

```shell
[root@localhost ~]# id 用户名
```

## 16、su命令：用户间切换（包含su和su -的区别）

su 是最简单的用户切换命令，通过该命令可以实现任何身份的切换，包括从普通用户切换为 root 用户、从 root 用户切换为普通用户以及普通用户之间的切换。

<!--普通用户之间切换以及普通用户切换至 root 用户，都需要知晓对方的密码，只有正确输入密码，才能实现切换；从 root 用户切换至其他用户，无需知晓对方密码，直接可切换成功。-->

su 命令的基本格式如下：

```shell
[root@localhost ~]# su [选项] 用户名
```


选项：

- -：当前用户不仅切换为指定用户的身份，同时所用的工作环境也切换为此用户的环境（包括 PATH 变量、MAIL 变量等），使用 - 选项可省略用户名，默认会切换为 root 用户。
- -l：同 - 的使用类似，也就是在切换用户身份的同时，完整切换工作环境，但后面需要添加欲切换的使用者账号。
- -p：表示切换为指定用户的身份，但不改变当前的工作环境（不使用切换用户的配置文件）。
- -m：和 -p 一样；
- -c 命令：仅切换用户执行一次命令，执行后自动切换回来，该选项后通常会带有要执行的命令。

### su 和 su - 的区别

注意，使用 su 命令时，有 - 和没有 - 是完全不同的，- 选项表示在切换用户身份的同时，连当前使用的环境变量也切换成指定用户的。我们知道，环境变量是用来定义操作系统环境的，因此如果系统环境没有随用户身份切换，很多命令无法正确执行。

举个例子，普通用户 lamp 通过 su 命令切换成 root 用户，但没有使用 - 选项，这样情况下，虽然看似是 root 用户，但系统中的 $PATH 环境变量依然是 lamp 的（而不是 root 的），因此当前工作环境中，并不包含 /sbin、/usr/sbin等超级用户命令的保存路径，这就导致很多管理员命令根本无法使用。不仅如此，当 root 用户接受邮件时，会发现收到的是 lamp 用户的邮件，因为环境变量 $MAIL 也没有切换。

## 17、whoami和who am i命令用法和区别

whoami 命令和 who am i 命令是不同的 2 个命令，前者用来打印当前执行操作的用户名，后者则用来打印登陆当前 Linux 系统的用户名。

例如：

```shell
[root@Test-BK-Dev ~]# su - test
Last login: Fri Jul  9 14:19:50 CST 2021 on pts/0
[test@Test-BK-Dev ~]$ whoami
test
[test@Test-BK-Dev ~]$ who am i
root     pts/0        2021-07-09 10:40 (10.6.115.133)
```

使用 su 或者 sudo 命令切换用户身份，骗得过 whoami，但骗不过 who am i。要解释这背后的运行机制，需要搞清楚什么是实际用户（UID）和有效用户（EUID，即 Effective UID）。

那么，whoami 和 who am i通常应用在哪些场景中呢？通常，对那些经常需要切换用户的系统管理员来说，经常需要明确当前使用的是什么身份；另外，对于某些 shell 脚本，或者需要特别的用户才能执行，这时就需要利用 whoami 命令来搞清楚执行它的用户是谁；甚至还有一些 shell 脚本，一定要某个特别用户才能执行，即便使用 su 或者 sudo 命令切换到此身份都不行，此时就需要利用 who am i 来确认。

<!--使用 su 或者 sudo 命令切换用户身份，骗得过 whoami，但骗不过 who am i。要解释这背后的运行机制，需要搞清楚什么是实际用户（UID）和有效用户（EUID，即 Effective UID）。-->

<!--所谓实际用户，指的是登陆 Linux 系统时所使用的用户，因此在整个登陆会话过程中，实际用户是不会发生变化的；而有效用户，指的是当前执行操作的用户，也就是说真正决定权限高低的用户，这个是能够利用 su 或者 sudo 命令进行任意切换的。-->

## 18、groupadd命令：添加用户组

添加用户组的命令是 groupadd，命令格式如下:

```shell
[root@localhost ~]# groupadd [选项] 组名
```

选项：

- -g GID：指定组 ID；
- -r：创建系统群组。

## 19、groupmod命令详解：修改用户组

groupmod 命令用于修改用户组的相关信息，命令格式如下：

```shell
[root@localhost ~]# groupmod [选现] 组名
```

选项：

- -g GID：修改组 ID；
- -n 新组名：修改组名；

不过大家还是要注意，用户名不要随意修改，组名和 GID 也不要随意修改，因为非常容易导致管理员逻辑混乱。如果非要修改用户名或组名，则建议大家先删除旧的，再建立新的。

## 20、groupdel命令：刪除用户组

groupdel 命令用于删除用户组（群组），此命令基本格式为：

```shell
[root@localhost ~]#groupdel 组名
```

通过前面的学习不难猜测出，使用 groupdel 命令删除群组，其实就是删除 /etc/gourp 文件和 /etc/gshadow 文件中有关目标群组的数据信息。

<!--注意，不能使用 groupdel 命令随意删除群组。此命令仅适用于删除那些 "不是任何用户初始组" 的群组，换句话说，如果有群组还是某用户的初始群组，则无法使用 groupdel 命令成功删除。-->

如果一定要删除某个初始群组，要么修改该组下用户的 GID，也就是将其初始组改为其他群组，要么先删除该初始组下的用户。

切记，虽然我们已经学了如何手动删除群组数据，但胡乱地删除群组可能会给其他用户造成不小的麻烦，因此更改文件数据要格外慎重。

## 21、gpasswd命令用法详解：把用户添加进组或从组中删除

我们可以使用 gpasswd 命令给群组设置一个群组管理员，代替 root 完成将用户加入或移出群组的操作。

gpasswd 命令的基本格式如下：

```shell
[root@localhost ~]# gpasswd 选项 组名
```

下表详细介绍了此命令提供的各种选项以及功能。

| 选项         | 功能                                                         |
| ------------ | ------------------------------------------------------------ |
|              | 选项为空时，表示给群组设置密码，仅 root 用户可用。           |
| -A user1,... | 将群组的控制权交给 user1,... 等用户管理，也就是说，设置 user1,... 等用户为群组的管理员，仅 root 用户可用。 |
| -M user1,... | 将 user1,... 加入到此群组中，仅 root 用户可用。              |
| -r           | 移除群组的密码，仅 root 用户可用。                           |
| -R           | 让群组的密码失效，仅 root 用户可用。                         |
| -a user      | 将 user 用户加入到群组中。                                   |
| -d user      | 将 user 用户从群组中移除。                                   |


从上表可以看到，除 root 可以管理群组外，可设置多个普通用户作为群组的管理员，但也只能做“将用户加入群组”和“将用户移出群组”的操作。

前面讲过，使用 `usermod -G` 命令也可以将用户加入群组，但会产生一个问题，即使用此命令将用户加入到新的群组后，会从之前加入的群组中踢出。

## 22、newgrp命令用法详解：切换用户的有效组

newgrp 命令可以从用户的附加组中选择一个群组，作为用户新的初始组。此命令的基本格式如下：

```shell
[root@localhost ~]# newgrp 组名
```

通过使用 newgrp 命令切换用户的初始组，所创建的文件各自属于不同的群组，这就是 newgrp 所发挥的作用，即通过切换附加组成为新的初始组，从而让用户获得使用各个附加组的权限。

# Linux权限管理



## 1、chgrp命令：修改文件和目录的所属组

chgrp使用基本格式：

```shell
 chgrp [-R] 所属组 文件名（目录名）
```

-R（注意是大写）选项长作用于更改目录的所属组，表示更改连同子目录中所有文件的所属组信息。

## 2、chown命令：修改文件和目录的所有者和所属组

chown使用基本格式：

当只需要修改所有者时：

```shell
chown [-R] 所有者 文件或目录
```

如果需要同时更改所有者和所属组时：

```shell
chown [-R] 所有者:所属组 文件或目录
```

注意，在 chown 命令中，所有者和所属组中间也可以使用点（.），但会产生一个问题，如果用户在设定账号时加入了小数点（例如 zhangsan.temp），就会造成系统误判。因此，建议大家使用冒号连接所有者和所属组。

当然，chown 命令也支持单纯的修改文件或目录的所属组，例如 

```shell
chown :group install.log
```

 就表示修改 install.log 文件的所属组，但修改所属组通常使用 chgrp 命令，因此并不推荐大家使用 chown 命令。

Linux 系统中，用户等级权限的划分是非常清楚的，root 用户拥有最高权限，可以修改任何文件的权限，而普通用户只能修改自己文件的权限（所有者是自己的文件）

## 3、Linux权限位

Linux 系统，最常见的文件权限有 3 种，即对文件的读（用 r 表示）、写（用 w 表示）和执行（用 x 表示，针对可执行文件或目录）权限。在 Linux 系统中，每个文件都明确规定了不同身份用户的访问权限，通过 ls 命令即可看到。

除此之外，我们有时会看到 s（针对可执行文件或目录，使文件在执行阶段，临时拥有文件所有者的权限）和 t（针对目录，任何用户都可以在此目录中创建文件，但只能删除自己的文件），文件设置 s 和 t 权限，会占用 x 权限的位置。

```shell
[root@localhost ~]# ls -al
total 156
drwxr-x---.   4    root   root     4096   Sep  8 14:06 .
drwxr-xr-x.  23    root   root     4096   Sep  8 14:21 ..
-rw-------.   1    root   root     1474   Sep  4 18:27 anaconda-ks.cfg
-rw-------.   1    root   root      199   Sep  8 17:14 .bash_history
-rw-r--r--.   1    root   root       24   Jan  6  2007 .bash_logout
...
```

可以看到，每行的第一列表示的就是各文件针对不同用户设定的权限，一共 11 位，但第 1 位用于表示文件的具体类型，最后一位此文件受 SELinux 的安全规则管理，不是本节关心的内容，放到后续章节做详细介绍。

```shell
-rwxr-xr--  root root    #没有selinux上下文，没有ACL
-rwx--xr-x+ root root    #只有ACL，没有selinux上下文
-rw-r--r--. root root    #只有selinux上下文，没有ACL
-rwxrwxr--+ root root    #有selinux上下文，有ACL
```

因此，为文件设定不同用户的读、写和执行权限，仅涉及到 9 位字符，以 ls 命令输出信息中的 .bash_logout 文件为例，设定不同用户的访问权限是 rw-r--r--，Linux 将访问文件的用户分为 3 类，分别是文件的所有者，所属组（也就是文件所属的群组）以及其他人。

有关群组的概念，我们已在用户和用户组一章中做了说明。除了所有者，以及所属群组中的用户可以访问文件外，其他用户（其他群组中的用户）也可以访问文件，这部分用户都归为其他人范畴。

很显然，Linux 系统为 3 种不同的用户身份，分别规定了是否对文件有读、写和执行权限。拿图 1 来说，文件所有者拥有对文件的读和写权限，但是没有执行权限；所属群组中的用户只拥有读权限，也就是说，这部分用户只能读取文件内容，无法修改文件；其他人也是只能读取文件。

## 4、读写执行权限（-r、-w、-x）的真正含义

通过前面的学习，我们知道了给文件设定权限的重要性，也知道了如何给文件设定权限。那么，读（r）、写（w）、执行（x）权限到底指的是什么呢？

首先要告诉大家的是，这些权限的含义并没有表面上那么简单，甚至同一权限对文件和目录的含义也不相同。

### a）rwx 权限对文件的作用

文件，是系统中用来存储数据的，包括普通的文本文件、数据库文件、二进制可执行文件，等等。不同的权限对文件的含义如下表所示。

| rwx 权限      | 对文件的作用                                                 |
| ------------- | ------------------------------------------------------------ |
| 读权限（r）   | 表示可读取此文件中的实际内容，例如，可以对文件执行 cat、more、less、head、tail 等文件查看命令。 |
| 写权限（w）   | 表示可以编辑、新增或者修改文件中的内容，例如，可以对文件执行 vim、echo 等修改文件数据的命令。注意，无权限不赋予用户删除文件的权利，除非用户对文件的上级目录拥有写权限才可以。 |
| 执行权限（x） | 表示该文件具有被系统执行的权限。Window系统中查看一个文件是否为可执行文件，是通过扩展名（.exe、.bat 等），但在 Linux 系统中，文件是否能被执行，是通过看此文件是否具有 x 权限来决定的。也就是说，只要文件拥有 x 权限，则此文件就是可执行文件。但是，文件到底能够正确运行，还要看文件中的代码是否正确。 |


对于文件来说，执行权限是最高权限。给用户或群组设定权限时，是否赋予执行权限需要慎重考虑，否则会对系统安装造成严重影响。

### b）rwx 权限对目录的作用

目录，主要用来记录文件名列表，不同的权限对目录的作用如下表 所示。

| rwx    权限        | 对目录的作用                                                 |
| ------------------ | ------------------------------------------------------------ |
| 读权限       （r） | 表示具有读取目录结构列表的权限，也就是说，可以看到目录中有哪些文件和子目录。一旦对目录拥有 r 权限，就可以在此目录下执行 ls 命令，查看目录中的内容。 |
| 写权限    （w）    | 对于目录来说，w 权限是最高权限。对目录拥有 w 权限，表示可以对目录做以下操作：在此目录中建立新的文件或子目录；删除已存在的文件和目录（无论子文件或子目录的权限是怎样的）；对已存在的文件或目录做更名操作；移动此目录下的文件和目录的位置。一旦对目录拥有 w 权限，就可以在目录下执行 touch、rm、cp、mv 等命令。 |
| 执行权限     （x） | 目录是不能直接运行的，对目录赋予 x 权限，代表用户可以进入目录，也就是说，赋予 x 权限的用户或群组可以使用 cd 命令。 |


对目录来说，如果只赋予 r 权限，则此目录是无法使用的。很简单，只有 r 权限的目录，用户只能查看目录结构，根本无法进入目录（需要用 x 权限），更不用说使用了。

因此，对于目录来说，常用来设定目录的权限其实只有 0（---）、5（r-x）、7（rwx）这 3 种。

## 5、chmod命令：修改文件或目录的权限

chmod 命令设定文件权限的方式有 2 种，分别可以使用数字或者符号来进行权限的变更。

### a）chmod命令使用数字修改文件权限

Linux 系统中，文件的基本权限由 9 个字符组成，以 rwxrw-r-x 为例，我们可以使用数字来代表各个权限，各个权限与数字的对应关系如下：

```
r --> 4
w --> 2
x --> 1
```

由于这 9 个字符分属 3 类用户，因此每种用户身份包含 3 个权限（r、w、x），通过将 3 个权限对应的数字累加，最终得到的值即可作为每种用户所具有的权限。

拿 rwxrw-r-x 来说，所有者、所属组和其他人分别对应的权限值为：

```
所有者 = rwx = 4+2+1 = 7
所属组 = rw- = 4+2 = 6
其他人 = r-x = 4+1 = 5
```

所以，此权限对应的权限值就是 765。

使用数字修改文件权限的 chmod 命令基本格式为：

```shell
chmod [-R] 权限值 文件名
```

-R（注意是大写）选项表示连同子目录中的所有文件，也都修改设定的权限。

### b）chmod命令使用字母修改文件权限

既然文件的基本权限就是 3 种用户身份（所有者、所属组和其他人）搭配 3 种权限（rwx），chmod 命令中用 u、g、o 分别代表 3 种身份，还用 a 表示全部的身份（all 的缩写）。另外，chmod 命令仍使用 r、w、x 分别表示读、写、执行权限。

使用字母修改文件权限的 chmod 命令，其基本格式如下图所示。

![](http://c.biancheng.net/uploads/allimg/190417/2-1Z41G31209649.gif)

例如，如果我们要设定 .bashrc 文件的权限为 rwxr-xr-x，则可执行如下命令：

```shell
[root@localhost ~]# chmod u=rwx,go=rx .bashrc
[root@localhost ~]# ls -al .bashrc
-rwxr-xr-x. 1 root root 176 Sep 22 2004 .bashrc
```

再举个例子，如果想要增加 .bashrc 文件的每种用户都可做写操作的权限，可以使用如下命令：

```shell
[root@localhost ~]# ls -al .bashrc
-rwxr-xr-x. 1 root root 176 Sep 22 2004 .bashrc
[root@localhost ~]# chmod a+w .bashrc
[root@localhost ~]# ls -al .bashrc
-rwxrwxrwx. 1 root root 176 Sep 22 2004 .bashrc
```

## 6、umask详解：令新建文件和目录拥有默认权限

Windows 系统中，新建的文件和目录时通过继承上级目录的权限获得的初始权限，而 Linux 不同，它是通过使用 umask 默认权限来给所有新建的文件和目录赋予初始权限的。

```shell
[root@localhost ~]# umask
0022
#root用户默认是0022，普通用户默认是 0002
```

umask 默认权限由 4 个八进制数组成，但第 1 个数代表的是文件所具有的特殊权限（SetUID、SetGID、Sticky BIT），此部分内容放到后续章节中讲解，现在先不讨论。也就是说，后 3 位数字 "022" 才是本节真正要用到的 umask 权限值，将其转变为字母形式为 ----w--w-。

注意，虽然 umask 默认权限是用来设定文件或目录的初始权限，但并不是直接将 umask 默认权限作为文件或目录的初始权限，还要对其进行 "再加工"。

文件和目录的真正初始权限，可通过以下的计算得到：

```
文件（或目录）的初始权限 = 文件（或目录）的最大默认权限 - umask权限
```

如果按照官方的标准算法，需要将 umask 默认权限使用二进制并经过逻辑与和逻辑非运算后，才能得到最终文件或目录的初始权限，计算过程比较复杂，且容易出错，这里给出一个比较简单的计算方式。

显然，如果想最终得到文件或目录的初始权限值，我们还需要了解文件和目录的最大默认权限值。在 Linux 系统中，文件和目录的最大默认权限是不一样的：

- 对文件来讲，其可拥有的最大默认权限是 666，即 rw-rw-rw-。也就是说，使用文件的任何用户都没有执行（x）权限。原因很简单，执行权限是文件的最高权限，赋予时绝对要慎重，因此绝不能在新建文件的时候就默认赋予，只能通过用户手工赋予。
- 对目录来讲，其可拥有的最大默认权限是 777，即 rwxrwxrwx。


接下来，我们利用字母权限的方式计算文件或目录的初始权限。以 umask 值为 022 为例，分别计算新建文件和目录的初始权限：

- 文件的最大默认权限是 666，换算成字母就是 "-rw-rw-rw-"，umask 的值是 022，换算成字母为 "-----w--w-"。把两个字母权限相减，得到 (-rw-rw-rw-) - (-----w--w-) = (-rw-r--r--)，这就是新建文件的初始权限。

<!--注意，在计算文件或目录的初始权限时，不能直接使用最大默认权限和 umask 权限的数字形式做减法，这是不对的。例如，若 umask 默认权限的值为 033，按照数字形式计算文件的初始权限，666-033=633，但我们按照字母的形式计算会得到 （rw-rw-rw-) - (----wx-wx) = (rw-r--r--)，换算成数字形式是 644。-->

## umask默认权限的修改方法

umask 权限值可以通过如下命令直接修改：

```shell
[root@localhost ~]# umask 002
[root@localhost ~]# umask
0002
[root@localhost ~]# umask 033
[root@localhost ~]# umask
0033
```


不过，这种方式修改的 umask 只是临时有效，一旦重启或重新登陆系统，就会失效。如果想让修改永久生效，则需要修改对应的环境变量配置文件 /etc/profile。例如：

```shell
[root@localhost ~]# vim /etc/profile
...省略部分内容...
if [ $UID -gt 199]&&[ "'id -gn'" = "'id -un'" ]; then
    umask 002
    #如果UID大于199（普通用户），则使用此umask值
else
    umask 022
    #如果UID小于199（超级用户），则使用此umask值
fi
…省略部分内容…
```

## 7、ACL权限设置（setfacl和getfacl）

设定 ACl 权限，常用命令有 2 个，分别是 setfacl 和 getfacl 命令，前者用于给指定文件或目录设定 ACL 权限，后者用于查看是否配置成功。

getfacl 命令用于查看文件或目录当前设定的 ACL 权限信息。该命令的基本格式为：

```shell
[root@localhost ~]# getfacl 文件名
```

getfacl 命令的使用非常简单，且常和 setfacl 命令一起搭配使用。

setfacl 命令可直接设定用户或群组对指定文件的访问权限。此命令的基本格式为：

```shell
[root@localhost ~]# setfacl 选项 文件名
```

下表罗列出了该命令可以使用的所用选项及功能：

| 选          项 | 功能                                                         |
| -------------- | :----------------------------------------------------------- |
| -   m  参 数   | 设定 ACL 权限。如果是给予用户 ACL 权限，参数则使用 "u:用户名:权限" 的格式，例如 `setfacl -m u:st:rx /project` 表示设定 st 用户对 project 目录具有 rx 权限；如果是给予组 ACL 权限，参数则使用 "g:组名:权限" 格式，例如 `setfacl -m g:tgroup:rx /project` 表示设定群组 tgroup 对 project 目录具有 rx 权限。 |
| -x 参数        | 删除指定用户（参数使用 u:用户名）或群组（参数使用 g:群组名）的 ACL 权限，例如 `setfacl -x u:st /project` 表示删除 st 用户对 project 目录的 ACL 权限。 |
| -b             | 删除所有的 ACL 权限，例如 `setfacl -b /project` 表示删除有关 project 目录的所有 ACL 权限。 |
| -d             | 设定默认 ACL 权限，命令格式为 "setfacl -m d:u:用户名:权限 文件名"（如果是群组，则使用 d:g:群组名:权限），只对目录生效，指目录中新建立的文件拥有此默认权限，例如 `setfacl -m d:u:st:rx /project` 表示 st 用户对 project 目录中新建立的文件拥有 rx 权限。 |
| -    R         | 递归设定 ACL 权限，指设定的 ACL 权限会对目录下的所有子文件生效，命令格式为 "setfacl -m u:用户名:权限 -R 文件名"（群组使用 g:群组名:权限），例如 `setfacl -m u:st:rx -R /project` 表示 st 用户对已存在于 project 目录中的子文件和子目录拥有 rx 权限。 |
| -k             | 删除默认 ACL 权限。                                          |

### setfacl -m：给用户或群组添加 ACL 权限

示例如下：给用户设置acl权限

```shell
[root@Test-BK-Dev download]# setfacl -m u:zhangjs:rx /download/test
[root@Test-BK-Dev download]# ll
total 662100
-rw-r--r--. 1 root root          271 Dec  9  2020 base.repo
-rw-r-x--x. 1 test zhangjs        25 Jul  8 14:54 do
-rw-r--r--. 1 root root           84 Dec  9  2020 epel.repo
-rw-r--r--. 1 root root     24692215 Feb 16  2020 filebeat-7.6.0-x86_64.rpm
-rw-r--r--. 1 root root    644916075 Sep  9  2019 mysql-5.7.27-linux-glibc2.12-x86_64.tar.gz
-rw-r--r--. 1 root root      7936660 Jul  1  2020 percona-xtrabackup-24-2.4.20-1.el7.x86_64.rpm
drwxr-xr-x+ 2 root root          100 Jul 12 09:03 test
-rw-r--r--. 1 root root       425912 Mar 23  2020 zabbix-agent-4.0.15-1.el7.x86_64.rpm
[root@Test-BK-Dev download]# getfacl /download/test/
getfacl: Removing leading '/' from absolute path names
# file: download/test/
# owner: root
# group: root
user::rwx   <--用户名栏是空的，说明是所有者的权限
user:zhangjs:r-x  <--用户zhangjs的权限
group::r-x   <--组名栏是空的，说明是所属组的权限
mask::r-x    <--mask权限
other::r-x   <--其他人的权限
```

<!--\#如果查询时会发现，在权限位后面多了一个"+"，表示此目录拥有ACL权限-->

给用户组添加设置acl权限：

```shell
[root@Test-BK-Dev download]# setfacl -m g:zhangjs:r /download/test/
[root@Test-BK-Dev download]# getfacl /download/test/
getfacl: Removing leading '/' from absolute path names
# file: download/test/
# owner: root
# group: root
user::rwx
user:zhangjs:r-x
group::r-x
group:zhangjs:r--    #用户组拥有了读的权限
mask::r-x
other::r-x
```

### setfacl -d：设定默认 ACL 权限

默认 ACL 权限的作用是，如果给父目录设定了默认 ACL 权限，那么父目录中所有新建的子文件都会继承父目录的 ACL 权限。需要注意的是，默认 ACL 权限只对目录生效。

<!--所以可以按照这个方法记忆参数是d-->

```shell
[root@Test-BK-Dev download]# setfacl -m d:u:test:rwx /download/test
[root@Test-BK-Dev download]# getfacl /download/test/
getfacl: Removing leading '/' from absolute path names
# file: download/test/
# owner: root
# group: root
user::rwx
user:zhangjs:r-x
user:test:r--
group::r-x
group:zhangjs:r--
mask::r-x
other::r-x
default:user::rwx
default:user:test:rwx
default:group::r-x
default:mask::rwx
default:other::r-x
```

```shell
[root@Test-BK-Dev test]# ll
total 630224
-rw-r--r--. 1 root root       568 Jul 12 09:23 1111
-rw-r--r--. 1 root root 644916075 Jul 12 09:03 mysql-5.7.27-linux-glibc2.12-x86_64.tar.gz
-rw-r--r--. 1 root root    425912 Jul 12 09:03 zabbix-agent-4.0.15-1.el7.x86_64.rpm
[root@Test-BK-Dev test]# touch 22222
[root@Test-BK-Dev test]# ll
total 630224
-rw-r--r--. 1 root root       568 Jul 12 09:23 1111
-rw-rw-r--+ 1 root root         0 Jul 12 09:41 22222
-rw-r--r--. 1 root root 644916075 Jul 12 09:03 mysql-5.7.27-linux-glibc2.12-x86_64.tar.gz
-rw-r--r--. 1 root root    425912 Jul 12 09:03 zabbix-agent-4.0.15-1.el7.x86_64.rpm

```

设置默认acl以后可以看到，新建的22222文件权限位最后是个＋了

### setfacl -R：设定递归 ACL 权限

递归 ACL 权限指的是父目录在设定 ACL 权限时，所有的子文件和子目录也会拥有相同的 ACL 权限。

<!--注意，默认 ACL 权限指的是针对父目录中后续建立的文件和目录会继承父目录的 ACL 权限；递归 ACL 权限指的是针对父目录中已经存在的所有子文件和子目录会继承父目录的 ACL 权限。-->

### setfacl -x：删除指定的 ACL 权限

```shell
[root@Test-BK-Dev test]# setfacl -x u:test /download/test/
[root@Test-BK-Dev test]# getfacl /download/test/
getfacl: Removing leading '/' from absolute path names
# file: download/test/
# owner: root
# group: root
user::rwx
user:zhangjs:r-x
group::r-x
group:zhangjs:r--
mask::r-x
other::r-x
default:user::rwx
default:user:zhangjs:r-x
default:user:test:rwx
default:group::r-x
default:mask::rwx
default:other::r-x
```

<!--注意，默认acl并没有被删除-->

```shell
[root@Test-BK-Dev test]# setfacl -x d:u:test /download/test/
[root@Test-BK-Dev test]# getfacl /download/test/
getfacl: Removing leading '/' from absolute path names
# file: download/test/
# owner: root
# group: root
user::rwx
user:zhangjs:r-x
group::r-x
group:zhangjs:r--
mask::r-x
other::r-x
default:user::rwx
default:user:zhangjs:r-x
default:group::r-x
default:mask::r-x
default:other::r-x
```

删除默认权限的时候也要添加-d参数

### setfacl -b：删除指定文件的所有 ACL 权限

```shell
[root@Test-BK-Dev test]# setfacl -b /download/test/
[root@Test-BK-Dev test]# getfacl /download/test/
getfacl: Removing leading '/' from absolute path names
# file: download/test/
# owner: root
# group: root
user::rwx
group::r-x
other::r-x
```

## 8、mask有效权限详解

对比添加 ACL 权限前后 getfacl 命令的输出信息，后者多了 2 行信息，一行是我们对 st 用户设定的 r-x 权限，另一行就是 mask 权限。

mask 权限，指的是用户或群组能拥有的最大 ACL 权限，也就是说，给用户或群组设定的 ACL 权限不能超过 mask 规定的权限范围，超出部分做无效处理。

我们一般不更改 mask 权限，只要赋予 mask 最大权限（也就是 rwx），则给用户或群组设定的 ACL 权限本身就是有效的。

## 9、SetUID（SUID）文件特殊权限用法详解

其实除了 rwx 权限，还会用到 s 权限，例如：

```shell
[root@Test-BK-Dev /]# ls -l /usr/bin/passwd 
-rwsr-xr-x. 1 root root 27856 Apr  1  2020 /usr/bin/passwd
```

可以看到，原本表示文件所有者权限中的 x 权限位，却出现了 s 权限，此种权限通常称为 SetUID，简称 SUID 特殊权限。

SUID 特殊权限仅适用于可执行文件，所具有的功能是，只要用户对设有 SUID 的文件有执行权限，那么当用户执行此文件时，会以文件所有者的身份去执行此文件，一旦文件执行结束，身份的切换也随之消失。

举一个例子，我们都知道，Linux 系统中所有用户的密码数据都记录在 /etc/shadow 这个文件中，通过 ll /etc/shadow 命令可以看到，此文件的权限是 0（---------），也就是说，普通用户对此文件没有任何操作权限。

这就会产生一个问题，为什么普通用户可以使用 passwd 命令修改自己的密码呢？

本节开头已经显示了 passwd 命令的权限配置，可以看到，此命令拥有 SUID 特殊权限，而且其他人对此文件也有执行权限，这就意味着，任何一个用户都可以用文件所有者，也就是 root 的身份去执行 passwd 命令。

<!--Linux 系统中，绝对多数命令的文件所有者默认都是 root。-->

换句话说，当普通用户使用 passwd 命令尝试更改自己的密码时，实际上是在以 root 的身份执行passwd命令，正因为 root 可以将密码写入 /etc/shadow 文件，所以普通用户也能做到。只不过，一旦命令执行完成，普通用户所具有的 root身份也随之消失。

做个测试：先把s权限去掉：

```shell
[root@Test-BK-Dev /]# chmod u-s /usr/bin/passwd 
[root@Test-BK-Dev /]# ls -l /usr/bin/passwd 
-rwxr-xr-x. 1 root root 27856 Apr  1  2020 /usr/bin/passwd
```

切换成test用户修改自己的密码：

```shell
[test@Test-BK-Dev /]$ passwd 
Changing password for user test.
Changing password for test.
(current) UNIX password: 
#看起来没有什么问题
passwd: Authentication token manipulation error 
#<--鉴定令牌操作错误
```

显然，虽然用户有执行 passwd 命令的权限，但无修改 /etc/shadow 文件的权限，因此最终密码修改失败。

测试完成后别忘记把s权限加上去：

```shell
[root@Test-BK-Dev /]# chmod u+s /usr/bin/passwd 
```

那么，普通用户可以使用 cat 命令查看 /etc/shadow 文件吗？答案的否定的，因为 cat 不具有 SUID 权限，因此普通用户在执行 cat /etc/shadow 命令时，无法以 root 的身份，只能以普通用户的身份，因此无法成功读取。

由此，我们可以总结出，SUID 特殊权限具有如下特点：

- 只有可执行文件才能设定 SetUID 权限，对目录设定 SUID，是无效的。
- 用户要对该文件拥有 x（执行）权限。
- 用户在执行该文件时，会以文件所有者的身份执行。
- SetUID 权限只在文件执行过程中有效，一旦执行完毕，身份的切换也随之消失。

<!--不要轻易设置SetUID（SUID）权限，否则会带来重大安全隐患！-->

如果我们手动给默认无 SetUID 权限的系统命令赋予 SetUID 权限，很多原本普通用户不能查看和修改的文件，竟然可以查看了，以 /etc/passwd 和 /etc/shadow 文件为例，普通用户也可以将自己的 UID 手动修改为 0，这意味着，此用户升级成为了超级用户。除此之外，普通用户还可以修改例如 /etc/inittab 和 /etc/fstab 这样重要的系统文件，可以轻易地使系统瘫痪。

其实，任何只有管理员可以执行的命令，如果被赋予了 SetUID 权限，那么后果都是灾难性的。普通用户可以随时重启服务器、随时关闭看得不顺眼的服务、随时添加其他普通用户的服务器，可以想象是什么样子。所以，SetUID 权限不能随便设置。

如何防止他人（例如黑客）对 SetUID 权限的恶意篡改呢？这里，给大家提供以下几点建议：

1. 关键目录要严格控制写权限，比如 "/"、"/usr" 等。
2. 用户的密码设置要严格遵守密码规范。
3. 对系统中默认应该有 SetUID 权限的文件制作一张列表，定时检査有没有列表之外的文件被设置了 SetUID 权限。

前面 2 点不再做过多解释，这里就最后一点，给大家提供一个脚本，仅供参考。

首先，在服务器第一次安装完成后，马上查找系统中所有拥有 SetUID 和 SetGID 权限的文件，把它们记录下来，作为扫描的参考模板。如果某次扫描的结果和本次保存下来的模板不一致，就说明有文件被修改了 SetUID 和 SetGID 权限。命令如下：

```shell
[root@Test-BK-Dev /]# find / -perm -4000 -o -perm -2000 > /download/suid
#-perm安装权限査找。-4000对应的是SetUID权限，-2000对应的是SetGID权限
#-o是逻辑或"or"的意思。并把命令搜索的结果放在/root/suid.list文件中
#接下来，只要定时扫描系统，然后和模板文件比对就可以了。
```

脚本：

```shell
#!/bin/bash
find / -perm -4000 -o -perm -2000 > /tmp/setuid.check
#搜索系统中所有拥有SetUID和SetGID权限的文件，并保存到临时目录中
for i in $(cat /tmp/setuid.check)
#循环，每次循环都取出临时文件中的文件名
do
    grep $i /root/suid.list > /dev/null
    #比对这个文件名是否在模板文件中
    if ["$?"!="o"]
    #检测测上一条命令的返回值，如果不为0，则证明上一条命令报错
    then
        echo "$i isn't in listfile! " >>/root/suid_log_$(date +%F)
        #如果文件名不在模板文件中，则输出错误信息，并把报错写入日志中
    fi
done
rm -rf/tmp/setuid.check
#删除临时文件
```

这个脚本成功的关键在于模板文件是否正常。所以一定要安装完系统就马上建立模板文件，并保证模板文件的安全。

注意，除非特殊情况，否则不要手工修改 SetUID 和 SetGID 权限，这样做非常不安全。而且就算我们做实验修改了 SetUID 和 SetGID 权限，也要马上修改回来，以免造成安全隐患。

## 10、SetGID（SGID）文件特殊权限用法详解

当 s 权限位于所属组的 x 权限位时，就被称为 SetGID，简称 SGID 特殊权限。

与 SUID 不同的是，SGID 既可以对文件进行配置，也可以对目录进行配置。

同 SUID 类似，对于文件来说，SGID 具有如下几个特点：

- SGID 只针对可执行文件有效，换句话说，只有可执行文件才可以被赋予 SGID 权限，普通文件赋予 SGID 没有意义。
- 用户需要对此可执行文件有 x 权限；
- 用户在执行具有 SGID 权限的可执行文件时，用户的群组身份会变为文件所属群组；
- SGID 权限赋予用户改变组身份的效果，只在可执行文件运行过程中有效；

<!--其实，SGID 和 SUID 的不同之处就在于，SUID 赋予用户的是文件所有者的权限，而 SGID 赋予用户的是文件所属组的权限，就这么简单。-->

当一个目录被赋予 SGID 权限后，进入此目录的普通用户，其有效群组会变为该目录的所属组，会就使得用户在创建文件（或目录）时，该文件（或目录）的所属组将不再是用户的所属组，而使用的是目录的所属组。

也就是说，只有当普通用户对具有 SGID 权限的目录有 rwx 权限时，SGID 的功能才能完全发挥。比如说，如果用户对该目录仅有 rx 权限，则用户进入此目录后，虽然其有效群组变为此目录的所属组，但由于没有 x 权限，用户无法在目录中创建文件或目录，SGID 权限也就无法发挥它的作用。

## 11、Stick BIT（SBIT）文件特殊权限用法详解

Sticky BIT，简称 SBIT 特殊权限，可意为粘着位、粘滞位、防删除位等。

SBIT 权限仅对目录有效，一旦目录设定了 SBIT 权限，则用户在此目录下创建的文件或目录，就只有自己和 root 才有权利修改或删除该文件。

也就是说，当甲用户以目录所属组或其他人的身份进入 A 目录时，如果甲对该目录有 w 权限，则表示对于 A 目录中任何用户创建的文件或子目录，甲都可以进行修改甚至删除等操作。但是，如果 A 目录设定有 SBIT 权限，那就大不一样啦，甲用户只能操作自己创建的文件或目录，而无法修改甚至删除其他用户创建的文件或目录。

```shell
[root@Test-BK-Dev /]# ll -d /tmp
drwxrwxrwt. 7 root root 93 Jul 12 03:44 /tmp
```

可以看到，在其他人身份的权限设定中，原来的 x 权限位被 t 权限占用了，这就表示此目录拥有 SBIT 权限。通过下面一系列的命令操作，我们来具体看看 SBIT 权限对 /tmp 目录的作用。

```shell
[root@Test-BK-Dev tmp]# su - test
Last login: Mon Jul 12 09:19:38 CST 2021 on pts/0
[test@Test-BK-Dev ~]$ cd /tmp/
[test@Test-BK-Dev tmp]$ touch 2222
[test@Test-BK-Dev tmp]$ ll
total 0
-rw-r--r--. 1 root root 0 Jul 12 11:08 1111
-rw-rw-r--. 1 test test 0 Jul 12 11:09 2222
[test@Test-BK-Dev tmp]$ exit
logout
[root@Test-BK-Dev tmp]# su - zhangjs
Last login: Thu Jul  8 14:48:11 CST 2021 on pts/0
Last failed login: Mon Jul 12 11:09:21 CST 2021 on pts/1
There was 1 failed login attempt since the last successful login.
[zhangjs@Test-BK-Dev ~]$ cd /tmp
[zhangjs@Test-BK-Dev tmp]$ ll
total 0
-rw-r--r--. 1 root root 0 Jul 12 11:08 1111
-rw-rw-r--. 1 test test 0 Jul 12 11:09 2222
[zhangjs@Test-BK-Dev tmp]$ rm -rf 2222
rm: cannot remove ‘2222’: Operation not permitted
```

可以看到，虽然 /tmp 目录的权限设定是 777，但由于其具有 SBIT 权限，因此 test 用户在此目录创建的文件2222，zhangjs 用户删除失败。这就是 SBIT 权限的作用。

## 12、Linux文件特殊权限（SUID、SGID和SBIT）的设置

首先我们有必要知道 SUID、SGID、SBIT 分别对应的数字，如下所示：

```
4 --> SUID
2 --> SGID
1 --> SBIT
```

如果要将一个文件权限设置为 -rwsr-xr-x，怎么办呢？此文件的普通权限为 755，另外，此文件还有 SUID 权限，因此只需在 755 的前面，加上 SUID 对应的数字 4 即可。也就是说，只需执行`chmod 4755 文件名`命令，就完成了-rwsr-xr-x 权限的设定。

<!--关于 -rwsr-xr-x 的普通权限是 755，你可以这样理解，标记有 s 和 t 的权限位，隐藏有 x 权限-->

同样的道理，如果某文件拥有 SUID 和 SGID 权限，则只需要给 chmod 命令传递 6---（- 表示数字）即可；如果某目录拥有 SGID 和 SBIT，只需要给 chmod 命令传递 3--- 即可。

注意，不同的特殊权限，作用的对象是不同的，SUID 只对可执行文件有效；SGID 对可执行文件和目录都有效；SBIT 只对目录有效。当然，你也可以给文件设置 7---，也就是将 SUID、SGID、SBIT赋予一个文件或目录，例如：

```shell
[root@localhost ~]# chmod 7777 ftest
\#一次赋予SetUID、SetGID和SBIT权限
[root@localhost ~]# ll ftest
-rwsrwsrwt. 1 root root Apr 19 23:54 ftest
```

执行过程虽然没有报错，但这样做，没有任何实际意义。

除了赋予 chmod 命令 4 个数字设定特殊权限，还可以使用字母的形式。例如，可以通过 "u+s" 给文件赋予 SUID 权限；通过 "g+s" 给文件或目录赋予 SGID 权限；通过 "o+t" 给目录赋予 SBIT 权限。

例子中，通过字母的形式成功给 ftest 文件赋予了 3 种特殊权限，此做法仅为验证字母形式的可行性，对 ftest 文件来说，并无实际意义。

细心的读者可能发现这样一个问题，使用 chmod 命令给文件或目录赋予特殊权限时，原文件或目录中存在的 x 权限会被替换成 s 或 t，而当我们使用 chmod 命令消除文件或目录的特殊权限时，原本消失的 x 权限又会显现出来。

这是因为，无论是 SUID、SGID 还是 SBIT，它们只针对具有 x 权限的文件或目录有效。没有 x 权限的文件或目录，即便赋予特殊权限，也无法发挥它们的功能，没有任何意义。

比如：

```shell
[root@Test-BK-Dev download]# touch 3333
[root@Test-BK-Dev download]# ll 3333
-rw-r--r--. 1 root root 0 Jul 12 11:18 3333     #新建的3333文件没有x权限
[root@Test-BK-Dev download]# chmod u+s,g+s,o+t 3333   #赋予s,s,t权限
[root@Test-BK-Dev download]# ll 3333
-rwSr-Sr-T. 1 root root 0 Jul 12 11:18 3333    
#可以看到，相应的权限位会被标记为 S（大写）和 T（大写），指的就是设置的 SUID、SGID 和 SBIT 权限没有意义。
```

## 13、chattr命令详解：修改文件系统的权限属性

管理 Linux 系统中的文件和目录，除了可以设定普通权限和特殊权限外，还可以利用文件和目录具有的一些隐藏属性。

chattr 命令，专门用来修改文件或目录的隐藏属性，只有 root 用户可以使用。该命令的基本格式为：

```shell
[root@localhost ~]# chattr [+-=] [属性] 文件或目录名
```

\+ 表示给文件或目录添加属性，- 表示移除文件或目录拥有的某些属性，= 表示给文件或目录设定一些属性。

下表 列出了常用的一些属性及功能。（字母 'aAcCdDeijsStTu' 可以赋予文件的新属性）

| 属性选项 | 功能                                                         |
| :------: | :----------------------------------------------------------- |
|    i     | 如果对文件设置 i 属性，那么不允许对文件进行删除、改名，也不能添加和修改数据； 如果对目录设置 i 属性，那么只能修改目录下文件中的数据，但不允许建立和删除文件； |
|    a     | 如果对文件设置 a 属性，那么只能在文件中増加数据，但是不能删除和修改数据； 如果对目录设置 a 属性，那么只允许在目录中建立和修改文件，但是不允许删除文件； |
|    u     | 设置此属性的文件或目录，在删除时，其内容会被保存，以保证后期能够恢复，常用来防止意外删除文件或目录。 |
|    s     | 和 u 相反，删除文件或目录时，会被彻底删除（直接从硬盘上删除，然后用 0 填充所占用的区域），不可恢复。 |

给设置有 i 属性的文件删除此属性也很简单，只需将 chattr 命令中 + 改为 - 即可。

##  14、lsattr命令：查看文件系统属性

使用 chattr 命令配置文件或目录的隐藏属性后，可以使用 lsattr 命令查看。

lsattr 命令，用于显示文件或目录的隐藏属性，其基本格式如下：

```shell
[root@localhost ~]# lsattr [选项] 文件或目录名
```

常用选项有以下 3 种：

- -a：后面不带文件或目录名，表示显示所有文件和目录（包括隐藏文件和目录）
- -d：如果目标是目录，只会列出目录本身的隐藏属性，而不会列出所含文件或子目录的隐藏属性信息；
- -R：和 -d 恰好相反，作用于目录时，会连同子目录的隐藏信息数据也一并显示出来。

```shell
[root@Test-BK-Dev download]# chattr +a do
[root@Test-BK-Dev download]# lsattr do
----ia---------- do
```

## 15、sudo命令用法详解：系统权限管理

我们知道，使用 su 命令可以让普通用户切换到 root 身份去执行某些特权命令，但存在一些问题，比如说：

- 仅仅为了一个特权操作就直接赋予普通用户控制系统的完整权限；
- 当多人使用同一台主机时，如果大家都要使用 su 命令切换到 root 身份，那势必就需要 root 的密码，这就导致很多人都知道 root 的密码；

考虑到使用 su 命令可能对系统安装造成的隐患，最常见的解决方法是使用 sudo 命令，此命令也可以让你切换至其他用户的身份去执行命令。

相对于使用 su 命令还需要新切换用户的密码，sudo 命令的运行只需要知道自己的密码即可，甚至于，我们可以通过手动修改 sudo 的配置文件，使其无需任何密码即可运行。

sudo 命令默认只有 root 用户可以运行，该命令的基本格式为：

```shell
[root@localhost ~]# sudo [-b] [-u 新使用者账号] 要执行的命令
```

常用的选项与参数：

- -b ：将后续的命令放到背景中让系统自行运行，不对当前的 shell 环境产生影响。
- -u ：后面可以接欲切换的用户名，若无此项则代表切换身份为 root 。
- -l：此选项的用法为 sudo -l，用于显示当前用户可以用 sudo 执行那些命令。

前面说过，默认情况下 sudo 命令只有 root 身份可以使用，那么，如何让普通用户也能使用它呢？

解决这个问题之前，先给大家分析一下 sudo 命令的执行过程。sudo命令的运行，需经历如下几步：

- 当用户运行 sudo 命令时，系统会先通过 /etc/sudoers 文件，验证该用户是否有运行 sudo 的权限；
- 确定用户具有使用 sudo 命令的权限后，还要让用户输入自己的密码进行确认。出于对系统安全性的考虑，如果用户在默认时间内（默认是 5 分钟）不使用 sudo 命令，此后使用时需要再次输入密码；
- 密码输入成功后，才会执行 sudo 命令后接的命令。

显然，能否使用 sudo 命令，取决于对 /etc/sudoers 文件的配置（默认情况下，此文件中只配置有 root 用户）。所以接下来，我们学习对 /etc/sudoers 文件进行合理的修改。

修改 /etc/sudoers，不建议直接使用 vim，而是使用 visudo。因为修改 /etc/sudoers 文件需遵循一定的语法规则，使用 visudo 的好处就在于，当修改完毕 /etc/sudoers 文件，离开修改页面时，系统会自行检验 /etc/sudoers 文件的语法。

修改其中：

```shell
[root@localhost ~]# visudo
…省略部分输出…
root ALL=(ALL) ALL  <--大约 76 行的位置
# %wheel ALL=(ALL) ALL   <--大约84行的位置
#这两行是系统为我们提供的模板，我们参照它写自己的就可以了
…省略部分输出…
```

这两行模板的含义分为是：

```
root ALL=(ALL) ALL
\#用户名 被管理主机的地址=(可使用的身份) 授权命令(绝对路径)
\#%wheel ALL=(ALL) ALL
\#%组名 被管理主机的地址=(可使用的身份) 授权命令(绝对路径)
```

下表 对以上 2 个模板的各部分进行详细的说明。

| 模块             | 含义                                                         |
| ---------------- | ------------------------------------------------------------ |
| 用户名或群组名   | 表示系统中的那个用户或群组，可以使用 sudo 这个命令。         |
| 被管理主机的地址 | 用户可以管理指定 IP 地址的服务器。这里如果写 ALL，则代表用户可以管理任何主机；如果写固定 IP，则代表用户可以管理指定的服务器。如果我们在这里写本机的 IP 地址，不代表只允许本机的用户使用指定命令，而是代表指定的用户可以从任何 IP 地址来管理当前服务器。 |
| 可使用的身份     | 就是把来源用户切换成什么身份使用，（ALL）代表可以切换成任意身份。这个字段可以省略。 |
| 授权命令         | 表示 root 把什么命令命令授权给用户，换句话说，可以用切换的身份执行什么命令。需要注意的是，此命令必须使用绝对路径写。默认值是 ALL，表示可以执行任何命令。 |

【例 1】
授权用户 lamp 可以重启服务器，由 root 用户添加，可以在 /etc/sudoers 模板下添加如下语句：

```shell
[root@localhost ~]# visudo
lamp ALL=/sbin/shutdown -r now
```

注意，这里也可以写多个授权命令，之间用逗号分隔。用户 lamp 可以使用 sudo -l 查看授权的命令列表：

```shell
[root@localhost ~]# su - lamp
\#切换成lamp用户
[lamp@localhost ~]$ sudo -l
[sudo] password for lamp:
\#需要输入lamp用户的密码
User lamp may run the following commands on this host:
(root) /sbin/shutdown -r now
```

可以看到，lamp 用户拥有了 shutdown -r now 的权限。这时，lamp 用户就可以使用 sudo 执行如下命令重启服务器：

```shell
[lamp@localhost ~]$ sudo /sbin/shutdown -r now
```

再次强调，授权命令要使用绝对路径（或者把 /sbin 路径导入普通用户 PATH 路径中，不推荐使用此方式），否则无法执行。

【例 2】
假设现在有 pro1，pro2，pro3 这 3 个用户，还有一个 group 群组，我们可以通过在 /etc/sudoers 文件配置 wheel 群组信息，令这 3 个用户同时拥有管理系统的权限。

首先，向 /etc/sudoers 文件中添加群组配置信息：

```shell
[root@localhost ~]# visudo
....(前面省略)....
%group   ALL=(ALL)  ALL
\#在 84 行#wheel这一行后面写入
```

此配置信息表示，group 这个群组中的所有用户都能够使用 sudo 切换任何身份，执行任何命令。接下来，我们使用 usermod 命令将 pro1 加入 group 群组，看看有什么效果：

```shell
[root@localhost ~]# usermod -a -G group pro1
[pro1@localhost ~]# sudo tail -n 1 /etc/shadow <==注意身份是 pro1
....(前面省略)....
Password: <==输入 pro1 的口令喔！
pro3:$1$GfinyJgZ$9J8IdrBXXMwZIauANg7tW0:14302:0:99999:7:::
[pro2@localhost ~]# sudo tail -n 1 /etc/shadow <==注意身份是 pro2
Password:
pro2 is not in the sudoers file. This incident will be reported.
\#此错误信息表示 pro2 不在 /etc/sudoers 的配置中。
```

可以看到，由于 pro1 加入到了 group 群组，因此 pro1 就可以使用 sudo 命令，而 pro2 不行。同样的道理，如果我们想让 pro3 也可以使用 sudo 命令，不用再修改 /etc/sudoers 文件，只需要将 pro3 加入 group 群组即可。

## 16、Linux权限对指令执行的影响

通过本章的学习我们知道，权限对于使用者账号是非常重要的，因为它可以限制使用者是否能读取、建立、删除、修改文件或目录。

本节将结合前面章节学到的有关文件系统管理的指令，通过几个实例向大家说明，权限对 Linux 指令执行的重要性。

【实例 1】
让当前用户进入某指定目录，可以使用什么指令？需要具备何种权限？

> 用户可以使用 cd 指令，同时要想使此命令成功执行，需要用户对要进入的目录具有 x 权限。另外，如果用户还想要在此目录中使用 ls 命令，还需要对此目录具有 r 权限。


【实例 2】
如果想在某目录内读取一个文件，可以使用什么指令？需要具备何种权限？

> 用户可以使用 cat、more、less 等指令，并且该用户对此目录至少需要具有 x 权限，对读取的文件需要具有 r 权限。



【实例 3】
如果想修改一个文件，可以使用什么指令？需要具备何种权限？

> 可以使用 vim 编辑器，对于权限方面，用户至少需要对该文件所在目录具有 x 权限，同时对该文件具有 r、w 权限。


【实例 4】
要想让使用者 Linuxer 能够执行 cp /dir1/file1 /dir2 的指令，则 Linuxer 需要对 dir1、file1、dir2 分别具备哪些权限。

> 执行 cp 命令时，Linuxer 要能够读取指定文件，并且能够写入目标文件，因此：
>
> - dir1：至少需要有 x 权限；
> - file1：至少需要有 r 权限；
> - dir2：至少需要有 w，x 权限。


【实例 5】
有一个文件，其绝对路径为 /home/student/www/index.html，其中各个相关文件或者目录的权限分别如下所示：

```shell
drwxr-xr-x 23  root   root 4096 Sep 22 12:09 /
drwxr-xr-x 6  root   root 4096 Sep 22 02:09 /home
drwx------ 6 student student 4096 Sep 22 02:10 /home/student
drwxr-xr-x 6 student student 4096 Sep 22 02:10 /home/student/www
drwxr--r-- 6 student student 369 Sep 22 02:11 /home/student/www/index.html
```

那么，当使用 test 这个账号（不属于 student 群组）能够成功读取 index.html 这个文件呢？

> 因为目录结构是由根目录一层一层读取的，通过分析以上各个目录和文件的权限得知，对于 test 账号来说，它可以进入 /home，但却不可以进入 /home/student，因此可以判定，test 无法成功读取 index.html 文件中的内容。

# 存储结构与管理硬盘

## 1、 Linux系统中常见的目录名称以及相应内容

| 目录名称    | 应放置文件的内容                                             |
| ----------- | ------------------------------------------------------------ |
| /boot       | 开机所需文件—内核、开机菜单以及所需配置文件等                |
| /dev        | 以文件形式存放任何设备与接口                                 |
| /etc        | 配置文件                                                     |
| /home       | 用户主目录                                                   |
| /bin        | 存放单用户模式下还可以操作的[命令](https://www.linuxcool.com/) |
| /lib        | 开机时用到的函数库，以及/bin与/sbin下面的命令要调用的函数    |
| /sbin       | 开机过程中需要的命令                                         |
| /media      | 用于挂载设备文件的目录                                       |
| /opt        | 放置第三方的软件                                             |
| /root       | 系统管理员的家目录                                           |
| /srv        | 一些网络服务的数据文件目录                                   |
| /tmp        | 任何人均可使用的“共享”临时目录                               |
| /proc       | 虚拟文件系统，例如系统内核、进程、外部设备及网络状态等       |
| /usr/local  | 用户自行安装的软件                                           |
| /usr/sbin   | Linux系统开机时不会使用到的软件/命令/[脚本](https://www.linuxcool.com/) |
| /usr/share  | 帮助与说明文件，也可放置共享文件                             |
| /var        | 主要存放经常变化的文件，如日志                               |
| /lost+found | 当文件系统发生错误时，将一些丢失的文件片段存放在这里         |

在Linux系统中一切都是文件，硬件设备也不例外。既然是文件，就必须有文件名称。系统内核中的udev设备管理器会自动把硬件名称规范起来，目的是让用户通过设备文件的名字可以猜出设备大致的属性以及分区信息等；这对于陌生的设备来说特别方便。另外，udev设备管理器的服务会一直以守护进程的形式运行并侦听内核发出的信号来管理/dev目录下的设备文件。Linux系统中常见的硬件设备的文件名称如表6-2所示。

## 2、常见的硬件设备及其文件名称

| 硬件设备      | 文件名称           |
| ------------- | ------------------ |
| IDE设备       | /dev/hd[a-d]       |
| SCSI/SATA/U盘 | /dev/sd[a-z]       |
| virtio设备    | /dev/vd[a-z]       |
| 软驱          | /dev/fd[0-1]       |
| 打印机        | /dev/lp[0-15]      |
| 光驱          | /dev/cdrom         |
| 鼠标          | /dev/mouse         |
| 磁带机        | /dev/st0或/dev/ht0 |

由于现在的IDE设备已经很少见了，所以一般的硬盘设备都会是以“/dev/sd”开头的。而一台主机上可以有多块硬盘，因此系统采用a～z来代表24块不同的硬盘（默认从a开始分配），而且硬盘的分区编号也很有讲究：

> <!--主分区或扩展分区的编号从1开始，到4结束；-->
>
> <!--逻辑分区从编号5开始。-->

<!--注意：，/dev目录中sda设备之所以是a，并不是由插槽决定的，而是由系统内核的识别顺序来决定的，而恰巧很多主板的插槽顺序就是系统内核的识别顺序，因此才会被命名为/dev/sda。大家以后在使用iSCSI网络存储设备时就会发现，明明主板上第二个插槽是空着的，但系统却能识别到/dev/sdb这个设备就是这个道理。-->



# Linux文件系统管理

## 1、硬盘结构

从存储数据的介质上来区分，硬盘可分为机械硬盘（Hard Disk Drive, HDD）和固态硬盘（Solid State Disk, SSD），机械硬盘采用磁性碟片来存储数据，而固态硬盘通过闪存颗粒来存储数据。

### A）机械硬盘   HDD

机械硬盘内部结构：

![](http://c.biancheng.net/uploads/allimg/181012/2-1Q012154JE59.jpg)

机械硬盘主要由磁盘盘片、磁头、主轴与传动轴等组成，数据就存放在磁盘盘片中。大家见过老式的留声机吗？留声机上使用的唱片和我们的磁盘盘片非常相似，只不过留声机只有一个磁头，而硬盘是上下双磁头，盘片在两个磁头中间高速旋转，类似下图。

![](http://c.biancheng.net/uploads/allimg/181012/2-1Q012154TC11.jpg)

也就是说，机械硬盘是上下盘面同时进数据读取的。而且机械硬盘的旋转速度要远高于唱片（目前机械硬盘的常见转速是 7200 r/min），所以机械硬盘在读取或写入数据时，非常害怕晃动和磕碰。另外，因为机械硬盘的超高转速，如果内部有灰尘，则会造成磁头或盘片的损坏，所以机械硬盘内部是封闭的，如果不是在无尘环境下，则禁止拆开机械硬盘。

我们已经知道数据是写入磁盘盘片的，那么数据是按照什么结构写入的呢？机械硬盘的逻辑结构主要分为磁道、扇区和拄面。我们来看看下面这张图。

![](http://c.biancheng.net/uploads/allimg/181012/2-1Q01215492WL.jpg)

什么是磁道呢？每个盘片都在逻辑上有很多的同心圆，最外面的同心圆就是 0 磁道。我们将每个同心圆称作磁道（注意，磁道只是逻辑结构，在盘面上并没有真正的同心圆）。硬盘的磁道密度非常高，通常一面上就有上千个磁道。但是相邻的磁道之间并不是紧挨着的，这是因为磁化单元相隔太近会相互产生影响。

那扇区又是十么呢？扇区其实是很形象的，大家都见过折叠的纸扇吧，纸扇打开后是半圆形或扇形的，不过这个扇形是由每个扇骨组合形成的。在磁盘上每个同心圆是磁道，从圆心向外呈放射状地产生分割线（扇骨），将每个磁道等分为若干弧段，每个弧段就是一个扇区。每个扇区的大小是固定的，为 512Byte。扇区也是磁盘的最小存储单位。

柱面又是什么呢？如果硬盘是由多个盘片组成的，每个盘面都被划分为数目相等的磁道，那么所有盘片都会从外向内进行磁道编号，最外侧的就是 0 磁道。具有相同编号的磁道会形成一个圆柱，这个圆柱就被称作磁盘的柱面，如下图所示。

![](http://c.biancheng.net/uploads/allimg/181012/2-1Q012155002627.jpg)

硬盘的大小是使用"磁头数 x 柱面数 x 扇区数 x 每个扇区的大小"这样的公式来计算的。其中，磁头数（Heads）表示硬盘共有几个磁头，也可以理解为硬盘有几个盘面，然后乘以 2；柱面数（Cylinders）表示硬盘每面盘片有几条磁道；扇区数（Sectors）表示每条磁道上有几个扇区；每个扇区的大小一般是 512Byte。

机械硬盘通过接口与计算机主板进行连接。硬盘的读取和写入速度与接口有很大关系。

常见的机械硬盘接口有以下几种：

IDE 硬盘接口（Integrated Drive Eectronics，并口，即电子集成驱动器）也称作 "ATA硬盘" 或 "PATA硬盘"，是早期机械硬盘的主要接口，ATA133 硬盘的理论速度可以达到 133MB/s（此速度为理论平均值），IDE 硬盘接口如下图所示

![](http://c.biancheng.net/uploads/allimg/181012/2-1Q01215521J45.jpg)

SATA 接口（Serial ATA，串口），是速度更高的硬盘标准，具备了更高的传输速度，并具备了更强的纠错能力。目前已经是 SATA 三代，理论传输速度达到 600MB/s（此速度为理论平均值）

![](http://c.biancheng.net/uploads/allimg/181012/2-1Q012155250523.jpg)

SCSI 接口（Small Computer System Interface，小型计算机系统接口），广泛应用在服务器上，具有应用范围广、多任务、带宽大、CPU 占用率低及热插拔等优点，理论传输速度达到 320MB/s

![](http://c.biancheng.net/uploads/allimg/181012/2-1Q01215531C45.jpg)

### B）固态硬盘（SSD）

固态硬盘和传统的机械硬盘最大的区别就是不再采用盘片进行数据存储，而采用存储芯片进行数据存储。固态硬盘的存储芯片主要分为两种：一种是采用闪存作为存储介质的；另一种是采用DRAM作为存储介质的。目前使用较多的主要是采用闪存作为存储介质的固态硬盘，

![](http://c.biancheng.net/uploads/allimg/181012/2-1Q012155343128.jpg)

固态硬盘和机械硬盘对比主要有以下一些特点，如下所示：

| 对比项目  | 固态硬盘        | 机械硬盘 |
| --------- | --------------- | -------- |
| 容量      | 较小            | 大       |
| 读/写速度 | 极快            | —般      |
| 写入次数  | 5000〜100000 次 | 没有限制 |
| 工作噪声  | 极低            | 有       |
| 工作温度  | 极低            | 较高     |
| 防震      | 很好            | 怕震动   |
| 重量      | 低              | 高       |
| 价格      | 高              | 低       |

大家可以发现，固态硬盘因为丟弃了机械硬盘的物理结构，所以相比机械硬盘具有了低能耗、无噪声、抗震动、低散热、体积小和速度快的优势；不过价格相比机械硬盘更高，而且使用寿命有限。

## 2、Linux常见的文件系统有哪些，CentOS采用哪种文件系统？

Linux 系统能够支持的文件系统非常多，除 Linux 默认文件系统 Ext2、Ext3 和 Ext4 之外，还能支持 fat16、fat32、NTFS(需要重新编译内核)等 Windows 文件系统。也就是说，Linux 可以通过挂载的方式使用 Windows 文件系统中的数据。Linux 所能够支持的文件系统在 "/usr/src/kemels/当前系统版本/fs" 目录中(需要在安装时选择)，该目录中的每个子目录都是一个可以识别的文件系统。

| Ext     | Linux 中最早的文件系统，由于在性能和兼容性上具有很多缺陷，现在已经很少使用 |
| ------- | ------------------------------------------------------------ |
| Ext2    | 是 Ext 文件系统的升级版本，Red Hat Linux 7.2 版本以前的系统默认都是 Ext2 文件系统。于 1993 年发布，支持最大 16TB 的分区和最大 2TB 的文件(1TB=1024GB=1024x1024KB) |
| Ext3    | 是 Ext2 文件系统的升级版本，最大的区别就是带日志功能，以便在系统突然停止时提高文件系统的可靠性。支持最大 16TB 的分区和最大 2TB 的文件 |
| Ext4    | 是 Ext3 文件系统的升级版。Ext4 在性能、伸缩性和可靠性方面进行了大量改进。Ext4 的变化可以说是翻天覆地的，比如向下兼容 Ext3、最大 1EB 文件系统和 16TB 文件、无限数量子目录、Extents 连续数据块 概念、多块分配、延迟分配、持久预分配、快速 FSCK、日志校验、无日志模式、在线碎片整理、inode 增强、默认启用 barrier 等。它是 CentOS 6.3 的默认文件系统 |
| xfs     | 被业界称为最先进、最具有可升级性的文件系统技术，由 SGI 公司设计，目前最新的 CentOS 7 版本默认使用的就是此文件系统。 |
| swap    | swap 是 Linux 中用于交换分区的文件系统(类似于 Windows 中的虚拟内存)，当内存不够用时，使用交换分区暂时替代内存。一般大小为内存的 2 倍，但是不要超过 2GB。它是 Linux 的必需分区 |
| NFS     | NFS 是网络文件系统(Network File System)的缩写，是用来实现不同主机之间文件共享的一种网络服务，本地主机可以通过挂载的方式使用远程共享的资源 |
| iso9660 | 光盘的标准文件系统。Linux 要想使用光盘，必须支持 iso9660 文件系统 |
| fat     | 就是 Windows 下的 fatl6 文件系统，在 Linux 中识别为 fat      |
| vfat    | 就是 Windows 下的 fat32 文件系统，在 Linux 中识别为 vfat。支持最大 32GB 的分区和最大 4GB 的文件 |
| NTFS    | 就是 Windows 下的 NTFS 文件系统，不过 Linux 默认是不能识别 NTFS 文件系统的，如果需要识别，则需要重新编译内核才能支持。它比 fat32 文件系统更加安全，速度更快，支持最大 2TB 的分区和最大 64GB 的文件 |
| ufs     | Sun 公司的操作系统 Solaris 和 SunOS 所采用的文件系统         |
| proc    | Linux 中基于内存的虚拟文件系统，用来管理内存存储目录 /proc   |
| sysfs   | 和 proc —样，也是基于内存的虚拟文件系统，用来管理内存存储目录 /sysfs |
| tmpfs   | 也是一种基于内存的虚拟文件系统，不过也可以使用 swap 交换分区 |

## 3、Linux系统是怎样识别硬盘设备和硬盘分区的？

Linux 系统初始化时，会根据 MBR 来识别硬盘设备。

MBR，全称 Master Boot Record，可译为硬盘主引导记录，占据硬盘 0 磁道的第一个扇区。MBR 中，包括用来载入操作系统的可执行代码，实际上，此可执行代码就是 MBR 中前 446 个字节的 boot loader 程序（引导加载程序），而在 boot loader 程序之后的 64 个（16×4）字节的空间，就是存储的分区表（Partition table）相关信息，MBR结构示意图：

![](http://c.biancheng.net/uploads/allimg/190517/2-1Z51GGI32E.gif)

在分区表（Partition table）中，主要存储的值息包括分区号（Partition id）、分区的起始磁柱和分区的磁柱数量。所以 Linux 操作系统在初始化时就可以根据分区表中以上 3 种信息来识别硬盘设备。其中，常见的分区号如下：

- 0x5（或 0xf）：可扩展分区（Extended partition）。
- 0x82：Linux 交换区（Swap partition）。
- 0x83：普通 Linux 分区（Linux partition）。
- 0x8e：Linux 逻辑卷管理分区（Linux LVM partition）。
- 0xfd：Linux 的 RAID 分区（Linux RAID auto partition）。


由于 MBR 留给分区表的磁盘空间只有 64 个字节，而每个分区表的大小为 16 个字节，所以在一个硬盘上最多可以划分出 4 个主分区。如果想要在一个硬盘上划分出 4 个以上的分区时，可以通过在硬盘上先划分出一个可扩展分区的方法来增加额外的分区。

不过，在 Linux 的 Kernel 中所支持的分区数量有如下限制：

- 一个 IDE 的硬盘最多可以使用 63 个分区；
- 一个 SCSI 的硬盘最多可以使用 15 个分区。


接下来的问题，就是为什么要将一个硬盘划分成多个分区，而不是直接使用整个硬盘呢？其主要有如下原因：

1. 方便管理和控制
   首先，可以将系统中的数据（也包括程序）按不同的应用分成几类，之后将这些不同类型的数据分别存放在不同的磁盘分区中。由于在每个分区上存放的都是类似的数据或程序，这样管理和维护就简单多了。
2. 提高系统的效率
   给硬盘分区，可以直接缩短系统读写磁盘时磁头移动的距离，也就是说，缩小了磁头搜寻的范围；反之，如果不使用分区，每次在硬盘上搜寻信息时可能要搜寻整个硬盘，所以速度会很慢。另外，硬盘分区也可以减轻碎片（文件不连续存放）所造成的系统效率下降的问题。
3. 使用磁盘配额的功能限制用户使用的磁盘量
   由于限制用户使用磁盘配额的功能，只能在分区一级上使用，所以，为了限制用户使用磁盘的总量，防止用户浪费磁盘空间（甚至将磁盘空间耗光），最好将磁盘先分区，然后在分配给一般用户。
4. 便于备份和恢复
   硬盘分区后，就可以只对所需的分区进行备份和恢复操作，这样的话，备份和恢复的数据量会大大地下降，而且也更简单和方便。

## 4、df用法详解：查看文件系统硬盘使用情况

df 命令，用于显示 Linux 系统中各文件系统的硬盘使用情况，包括文件系统所在硬盘分区的总容量、已使用的容量、剩余容量等。

前面讲过，与整个文件系统有关的数据，都保存在 Super block（超级块）中，而 df 命令主要读取的数据几乎都针对的是整个文件系统，所以 df 命令主要是从各文件系统的 Super block 中读取数据。

df 命令的基本格式为：

```shell
[root@localhost ~]# df [选项] [目录或文件名]
```

表 1 列出了 df 命令几个常用的选项，以及它们各自的作用。

| 选项 | 作用                                                         |
| ---- | ------------------------------------------------------------ |
| -a   | 显示所有文件系统信息，包括系统特有的 /proc、/sysfs 等文件系统； |
| -m   | 以 MB 为单位显示容量；                                       |
| -k   | 以 KB 为单位显示容量，默认以 KB 为单位；                     |
| -h   | 使用人们习惯的 KB、MB 或 GB 等单位自行显示容量；             |
| -T   | 显示该分区的文件系统名称；                                   |
| -i   | 不用硬盘容量显示，而是以含有 inode 的数量来显示。            |

```shell
[root@Test-BK-Dev /]# df
Filesystem              1K-blocks     Used Available Use% Mounted on
devtmpfs                  8103372        0   8103372   0% /dev
tmpfs                     8115144        0   8115144   0% /dev/shm
tmpfs                     8115144    91712   8023432   2% /run
tmpfs                     8115144        0   8115144   0% /sys/fs/cgroup
/dev/mapper/centos-root 192020884 12770108 179250776   7% /
/dev/sda1                  815780   192228    623552  24% /boot
tmpfs                     1623032        0   1623032   0% /run/user/0
```

本例中，由 df 命令显示出的各列信息的含义分别是：

- Filesystem：表示该文件系统位于哪个分区，因此该列显示的是设备名称；
- 1K-blocks：此列表示文件系统的总大小，默认以 KB 为单位；
- Used：表示用掉的硬盘空间大小；
- Available：表示剩余的硬盘空间大小；
- Use%：硬盘空间使用率。如果使用率高达 90% 以上，就需要额外注意，因为容量不足，会严重影响系统的正常运行；
- Mounted on：文件系统的挂载点，也就是硬盘挂载的目录位置。

## 5、du命令：统计目录或文件所占磁盘空间大小

du 命令的格式如下：

```shell
[root@localhost ~]# du [选项] [目录或文件名]
```

选项：

- -a：显示每个子文件的磁盘占用量。默认只统计子目录的磁盘占用量
- -h：使用习惯单位显示磁盘占用量，如 KB、MB 或 GB 等；
- -s：统计总磁盘占用量，而不列出子目录和子文件的磁盘占用量

## 6、mount命令详解：挂载Linux系统外的文件

挂载指的是将硬件设备的文件系统和 Linux 系统中的文件系统，通过指定目录（作为挂载点）进行关联。而要将文件系统挂载到 Linux 系统上，就需要使用 mount 挂载命令。

mount 命令的常用格式有以下几种：

```shell
[root@localhost ~]# mount [-l]
```

单纯使用 mount 命令，会显示出系统中已挂载的设备信息，使用 -l 选项，会额外显示出卷标名称（读者可自行运行，查看输出结果）；

```shell
[root@localhost ~]# mount -a
```

-a 选项的含义是自动检查 /etc/fstab 文件中有无疏漏被挂载的设备文件，如果有，则进行自动挂载操作。这里简单介绍一下 /etc/fstab 文件，此文件是自动挂载文件，系统开机时会主动读取 /etc/fstab 这个文件中的内容，根据该文件的配置，系统会自动挂载指定设备。

```shell
[root@localhost ~]# mount [-t 系统类型] [-L 卷标名] [-o 特殊选项] [-n] 设备文件名 挂载点
```

各选项的含义分别是：

- -t 系统类型：指定欲挂载的文件系统类型。Linux 常见的支持类型有 EXT2、EXT3、EXT4、iso9660（光盘格式）、vfat、reiserfs 等。如果不指定具体类型，挂载时 Linux 会自动检测。
- -L 卷标名：除了使用设备文件名（例如 /dev/hdc6）之外，还可以利用文件系统的卷标名称进行挂载。
- -n：在默认情况下，系统会将实际挂载的情况实时写入 /etc/mtab 文件中，但在某些场景下（例如单人维护模式），为了避免出现问题，会刻意不写入，此时就需要使用这个选项；
- -o 特殊选项：可以指定挂载的额外选项，比如读写权限、同步/异步等，如果不指定，则使用默认值（defaults）。具体的特殊选项参见表 1；

| 选项        | 功能                                                         |
| ----------- | ------------------------------------------------------------ |
| rw/ro       | 是否对挂载的文件系统拥有读写权限，rw 为默认值，表示拥有读写权限；ro 表示只读权限。 |
| async/sync  | 此文件系统是否使用同步写入（sync）或异步（async）的内存机制，默认为异步 async。 |
| dev/nodev   | 是否允许从该文件系统的 block 文件中提取数据，为了保证数据安装，默认是 nodev。 |
| auto/noauto | 是否允许此文件系统被以 mount -a 的方式进行自动挂载，默认是 auto。 |
| suid/nosuid | 设定文件系统是否拥有 SetUID 和 SetGID 权限，默认是拥有。     |
| exec/noexec | 设定在文件系统中是否允许执行可执行文件，默认是允许。         |
| user/nouser | 设定此文件系统是否允许让普通用户使用 mount 执行实现挂载，默认是不允许（nouser），仅有 root 可以。 |
| defaults    | 定义默认值，相当于 rw、suid、dev、exec、auto、nouser、async 这 7 个选项。 |
| remount     | 重新挂载已挂载的文件系统，一般用于指定修改特殊权限。         |

举例：

挂载：

```shell
mount -t nfs 10.6.22.162:/NFS/ECOLOGY /nfs
mount.nfs4 docnas.ht.com:/doctest/ /filesmnt/
```

## 7、Linux挂载光盘（使用mount命令）

将光盘放入光驱之后，需执行如下挂载命令：

```shell
[root@localhost ~]# mkdir/mnt/cdrom/
```

\#建立挂载点

```shell
[root@localhost ~]# mount -t iso9660 /dev/cdrom /mnt/cdrom/
```

\#挂载光盘

光盘的文件系统是 iso9660，不过这个文件系统可以省略不写，系统会自动检测。因此，挂在命令也可以写为如下的方式：

```shell
[root@localhost ~]# mount /dev/cdrom /mnt/cdrom/
\#挂载光盘。两个挂载光盘的命令使用一个就可以了
[root@localhost ~]# mount
\#查看已经挂载的设备
…省略部分输出…
/dev/srO on /mnt/cdrom type iso9660 (ro)
\#光盘已经挂载了，但是挂载的设备文件名是/dev/sr0
```

 /dev/cdrom 就是光驱的设备文件名，不过注意 /dev/cdrom 只是一个软链接（如同 Windows 系统中的文件快捷方式）。 命令如下：

```shell
[root@localhost ~]#ll /dev/cdrom
lrwxrwxrwx 1 root root 3 1月31 01:13/dev/cdrom ->sr0
```

/dev/cdrom 的源文件是 /dev/sr0。/dev/sr0 是光驱的真正设备文件名，代表 SCSI 接口或 SATA 接口的光驱，所以刚刚查询挂载时看到的光驱设备文件命令是 /dev/sr0。也就是说，挂载命令也可以写成这样：

```shell
[root@localhost ~]# mount /dev/sr0 /mnt/cdrom/
```

其实光驱的真正设备文件名是保存在 /proc/sys/dev/cdrom/info 文件中的，所以可以通过查看这个文件来查询光盘的真正设备文件名，命令如下：

```shell
[root@localhost ~]# cat /proc/sys/dev/cdrom/info
CD-ROM information, ld: cdrom.c 3.20 2003/12/17
drive name: sr0
…省略部分输出…
```

## 8、给Linux系统挂载U盘

挂载 U 盘和挂载光盘的方式是一样的，只不过光盘的设备文件名是固定的（/dev/sr0 或 /dev/cdrom），而 U 盘的设备文件名是在插入 U 盘后系统自动分配的。

因为 U 盘使用的是硬盘的设备文件名，而每台服务器上插入的硬盘数量和分区方式都是不一样的，所以 U 盘的设备号需要单独检测与分配，以免和硬盘的设备文件名产生冲突。

U 盘的设备文件名是系统自动分配的，我们只要查找出来然后挂载可以了。首先把 U 盘插入 Linux 系统中，这里需要注意的是，如果是虚拟机，则需要先把鼠标点入虚拟机再插入 U 盘。

通过fdisk -l 来查看U盘的设备名

查看到 U 盘的设备文件名，接下来就要创建挂载点了。命令如下：

```shell
[root@localhost ~]# mkdir /mnt/usb
```

然后就是挂载了，挂载命令如下：

```shell
[root@localhost ~]# mount -t vfat /dev/sdb1 /mnt/usb/
挂载U盘。因为是Windows分区，所以是vfat文件系统格式
[root@localhost ~]# cd /mnt/usb/
\#去挂载点访问U盘数据
[root@localhost usb]# ls
\#输出为乱码
\#之所以出现乱码，是因为编码格式不同
```

之所以出现乱码，是因为 U 盘是 Windows 中保存的数据，而 Windows 中的中文编码格式和 Linux 中的不一致，只需在挂载的时候指定正确的编码格式就可以解决乱码问题，命令如下：

```shell
[root@localhost ~]# mount -t vfat -o iocharset=utf8 /dev/sdb1 /mnt/usb/
\#挂载U盘，指定中文编码格式为UTF-8
[root@localhost ~]# cd /mnt/usb/
[root@localhost usb]# ls
1111111年度总结及计划表.xls ZsyqlHL7osKSPBoGshZBr6.mp4 协议书
12月21日.doc 恭喜发财(定）.mp4 新年VCR(定).mp4
\#可以正确地查看中文了
```

因为我们的 Linux 在安装时采用的是 UTF-8 编码格式，所以要让 U 盘在挂载时也指定为 UTF-8 编码格式，才能正确显示。

```shell
[root@localhost ~]# echo $LANG
zh_CN.UTF-8
\#查看一下Linux默认的编码格式
```

注意，Linux 默认是不支持 NTFS 文件系统的，所以默认是不能挂载 NTFS 格式的移动硬盘的。要想让 Linux 支持移动硬盘，主要有三种方法：

1. 重新编译内核，加入 ntfs 模块，然后安装 ntfs 模块即可；
2. 不自己编译内核，而是下载已经编译好的内核，直接安装即可；
3. 安装 NTFS 文件系统的第三方插件，也可以支持 NTFS 文件系统；

## 9、Linux自动挂载（配置/etc/fatab）详解

/etc/fatab用户Linux系统在启动的时候实现自动挂载硬盘或者其他设备

```shell
[root@Test-BK-Dev dev]# cat /etc/fstab 

#
# /etc/fstab
# Created by anaconda on Sat Aug 15 00:29:35 2020
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
/dev/mapper/centos-root /                       xfs     defaults        0 0
UUID=a2188c37-e5c4-4d7b-a350-ddb272b698cc /boot                   xfs     defaults        0 0
/dev/mapper/centos-swap swap                    swap    defaults        0 0
```

可以看到，在 fstab 文件中，每行数据都分为了 6 个字段，它们的含义分别是：

1. 用来挂载每个文件系统的分区设备文件名或 UUID（用于指代设备名）；
2. 挂载点；
3. 文件系统的类型；
4. 各种挂载参数；
5. 指定分区是否被 dump 备份；
6. 指定分区是否被 fsck 检测；

### /etc/fstab文件各字段的含义

1、UUID 即通用唯一标识符，是一个 128 位比特的数字，可以理解为就是硬盘的 ID，UUID 由系统自动生成和管理。

2、挂载点

3、第三个字段为文件系统名称，CentOS 6.3 的默认文件系统应该是 ext4。

4、第四个字段是挂载参数，这个参数和 mount 命令的挂载参数一致。

5、第五个字段表示“指定分区是否被 dump 备份”，0 代表不备份，1 代表备份，2 代表不定期备份。

6、第六个字段表示“指定分区是否被 fsck 检测”，0 代表不检测，其他数字代表检测的优先级，1 的优先级比 2 高。所以先检测 1 的分区，再检测 2 的分区。一般分区的优先级是 1，其他分区的优先级是 2。

## 10、修改/etc/fstab文件出错导致Linux不能启动，该怎么办？

　　1.启动linux提示失败，输入root账户密码，进入 repair filesystem#，注意此时修复fstab文件会提示readonly无法保存修改。

　　2.提权成root

　　3.mount / -o remount

　　这时候，/etc/fstab就可以修改了，这一步是核心内容

　　4.修改fstab文件 vi /etc/fstab

　　要是fstab文件坏了，有备份就还原一下，不行就自己写一个呗，关键得知道分区情况，以及以前挂在情况。

## 11、umount命令：卸载文件系统

umount 命令用于卸载已经挂载的硬件设备，该命令的基本格式如下：

```shell
[root@localhost ~]# umount 设备文件名或挂载点
```


注意，卸载命令后面既可以加设备文件名，也可以加挂载点，不过只能二选一，比如：

```shell
[root@localhost ~]# umount /mnt/usb
\#卸载U盘
[root@localhost ~]# umount /mnt/cdrom
\#卸载光盘
[root@localhost ~]# umount /dev/sr0
\#命令加设备文件名同样是可以卸载的
```

umount.nfs4: /gw-nfs: device is busy的处理方法：

umount -l /data_nas

## 12、fsck命令：检测和修复文件系统

计算机难免会由于某些系统因素或人为误操作（突然断电）出现系统异常，这种情况下非常容易造成文件系统的崩溃，严重时甚至会造成硬件损坏。这也是我们一直在强调的“服务器一定要先关闭服务再进行重启”的原因所在。

那么，如果真出现了文件系统损坏的情况，有办法修复吗？可以的，对于一些小问题，使用 fsck 命令就可以很好地解决。

fsck 命令用于检查文件系统并尝试修复出现的错误。该命令的基本格式如下：

```shell
[root@localhost ~]# fsck [选项] 分区设备文件名
```

| 选项            | 功能                                                         |
| --------------- | ------------------------------------------------------------ |
| -a              | 自动修复文件系统，没有任何提示信息。                         |
| -r              | 采取互动的修复模式，在修改文件前会进行询问，让用户得以确认并决定处理方式。 |
| -A（大写）      | 按照 /etc/fstab 配置文件的内容，检查文件内罗列的全部文件系统。 |
| -t 文件系统类型 | 指定要检查的文件系统类型。                                   |
| -C（大写）      | 显示检查分区的进度条。                                       |
| -f              | 强制检测，一般 fsck 命令如果没有发现分区有问题，则是不会检测的。如果强制检测，那么不管是否发现问题，都会检测。 |
| -y              | 自动修复，和 -a 作用一致，不过有些文件系统只支持 -y。        |

<!--此命令通常只有身为 root 用户且文件系统出现问题时才会使用，否则，在正常状况下使用 fsck 命令，很可能损坏系统。另外，如果你怀疑已经格式化成功的硬盘有问题，也可以使用此命令来进行检查。-->

<!--使用 fsck 检查并修复文件系统是存在风险的，特别是当硬盘错误非常严重的时候，因此，当一个受损文件系统中包含了非常有价值的数据时，务必首先进行备份！-->

<!--需要注意的是，在使用 fsck 命令修改某文件系统时，这个文件系统对应的磁盘分区一定要处于卸载状态，磁盘分区在挂载状态下进行修复是非常不安全的，数据可能会遭到破坏，也有可能会损坏磁盘。-->

这里，给大家举个例子，如果想要修复某个分区，则只需执行如下命令：

```shell
[root@localhost ~]#fsck -r /dev/sdb1
#采用互动的修复模式
```

fsck 命令在执行时，如果发现存在没有文件系统依赖的文件或目录，就会提示用户是否把它们找回来，因为这些没有文件系统依赖的文件或目录对用户来说是看不到的，换句话说，用户根本无法使用，这通常是由文件系统内部结构损坏导致的。如果用户同意找回（输入 y），fsck 命令就会把这些孤立的文件或目录放到 lost+found 目录中，并用这些文件自己对应的 inode 号来命名，以便用户查找自己丢失的文件。

因此，当用户在利用 fsck 命令修复磁盘分区以后，如果发现分区中有文件丢失，就可以到对应的 lost+found 目录中去查找，但由于无法通过文件名称分辨各个文件，这里可以利用 file 命令查看文件系统类型，进而判断出哪个是我们需要的文件。

##  13、fdisk命令详解：给硬盘分区


fdisk 命令的格式如下：

```shell
[root@localhost ~]# fdisk ~l
\#列出系统分区
[root@localhost ~]# fdisk 设备文件名
\#给硬盘分区
```

注意，千万不要在当前的硬盘上尝试使用 fdisk，这会完整删除整个系统，一定要再找一块硬盘，或者使用虚拟机

在 fdisk 交互界面中输入 m 可以得到帮助，帮助里列出了 fdisk 可以识别的交互命令，我们来解释一下这些命令：

| 命令 | 说 明                                                        |
| ---- | ------------------------------------------------------------ |
| a    | 设置可引导标记                                               |
| b    | 编辑 bsd 磁盘标签                                            |
| c    | 设置 DOS 操作系统兼容标记                                    |
| d    | 删除一个分区                                                 |
| 1    | 显示已知的文件系统类型。82 为 Linux swap 分区，83 为 Linux 分区 |
| m    | 显示帮助菜单                                                 |
| n    | 新建分区                                                     |
| 0    | 建立空白 DOS 分区表                                          |
| P    | 显示分区列表                                                 |
| q    | 不保存退出                                                   |
| s    | 新建空白 SUN 磁盘标签                                        |
| t    | 改变一个分区的系统 ID                                        |
| u    | 改变显示记录单位                                             |
| V    | 验证分区表                                                   |
| w    | 保存退出                                                     |
| X    | 附加功能（仅专家）                                           |

## 14、 fdisk创建分区（主分区、扩展分区和逻辑分区）过程详解

### A）fdisk创建主分区过程详解

```shell
[root@localhost ~]# fdisk /dev/sdb
…省略部分输出…
Command (m for help): p
#显示当前硬盘的分区列表
Disk /dev/sdb: 21.5 GB, 21474836480 bytes 255 heads, 63 sectors/track, 2610 cylinders
Units = cylinders of 16065 *512 = 8225280 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0xb4b0720c
Device Boot Start End Blocks id System
#目前一个分区都没有
Command (m for help): n
#那么我们新建一个分区
Command action
#指定分区类型
e extended
#扩展分区
p primary partition (1-4)
#主分区
p
#这里选择p，建立一个主分区
Partition number (1-4): 1
#选择分区号，范围为1~4，这里选择1
First cylinder (1 -2610, default 1):
#分区的起始柱面，默认从1开始。因为要从硬盘头开始分区，所以直接回车
Using default value 1
#提示使用的是默认值1
Last cylinder, +cylinders or +size{K, M, G}(1-2610, default 2610): +5G
#指定硬盘大小。可以按照柱面指定(1-2610)。我们对柱面不熟悉，那么可以使用size{K, M, G}的方式指定硬盘大小。这里指定+5G，建立一个5GB大小的分区
Command (m for help):
#主分区就建立了，又回到了fdisk交互界面的提示符
Command (m for help): p
#查询一下新建立的分区
Disk /dev/sdb: 21.5GB, 21474836480 bytes
255 heads,63 sectors/track, 2610 cylinders
Units = cylinders of 16065 * 512 = 8225280 bytes
Sector size (logical/physical): 512 bytes 1512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0xb4b0720c
Device Boot Start End Blocks id System
/dev/sdb1 1 654 5253223+ 83 Linux
#dev/sdb1已经建立了吧
```

总结，建立主分区的过程就是这样的："fdisk 硬盘名 -> n(新建)->p(建立主分区) -> 1(指定分区号) -> 回车（默认从 1 柱面开始建立分区）-> +5G(指定分区大小)"。当然，我们的分区还没有格式化和挂载，所以还不能使用。

### B）fdisk命令创建扩展分区过程详解

主分区和扩展分区加起来最多只能建立 4 个，而扩展分区最多只能建立 1 个。

```shell
[root@localhost ~]# fdisk /dev/sdb
…省略部分输出…
Command (m for help): n
#新建立分区
Command action
e extended
p primary partition (1-4)
e
#这次建立扩展分区
Partition number (1-4): 2
#给扩展分区指定分区号2
First cylinder (655-2610, default 655):
#扩展分区的起始柱面。上节建立的主分区1已经占用了1~654个柱面，所以我们从655开始建立，注意：如果没有特殊要求，则不要跳开柱面建立分区，应该紧挨着建立分区
Using default value 655
提示使用的是默认值655
Last cylinder, +cylinders or +size{K, M, G} (655-2610, default 2610):
#这里把整块硬盘的剩余空间都建立为扩展分区
Using default value 2610
#提示使用的是默认值2610
```

这里把 /dev/sdb 硬盘的所有剩余空间都建立为扩展分区，也就是建立一个主分区，剩余空间都建立成扩展分区，再由扩展分区中建立逻辑分区（逻辑分区在后续章节中介绍）。

### C）fdisk命令创建逻辑分区过程详解

扩展分区是不能被格式化和直接使用的，所以还要在扩展分区内部再建立逻辑分区。

我们来看看逻辑分区的建立过程，命令如下：

```shell
[root@localhost ~]# fdisk /dev/sdb
…省略部分输出…
Command (m for help): n
#建立新分区
Command action
l logical (5 or over)
#由于在前面章节中，扩展分区已经建立，所以这里变成了l(logic)
p primary partition (1-4)
l
#建立逻辑分区
First cylinder (655-2610, default 655):
#不用指定分区号，默认会从5开始分配，所以直接选择起始柱面
#注意：逻辑分区是在扩展分区内部再划分的，所以柱面是和扩展分区重叠的
Using default value 655
Last cylinder, +cylinders or +size{K, M, G} (655-2610, default 2610):+2G
#分配2GB大小
Command (m for help): n
#再建立一个逻辑分区
Command action
l logical (5 or over)
p primary partition (1-4)
l
First cylinder (917-2610, default 917):
Using default value 917
Last cylinder, +cylinders or +size{K, M, G} (917-2610, default 2610):+2G
Command (m for help): p
#查看一下已经建立的分区
Disk /dev/sdb: 21.5GB, 21474836480 bytes
255 heads, 63 sectors/track, 2610 cylinders
Units = cylinders of 16065 *512 = 8225280 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes Disk identifier: 0xb4b0720c
Device Boot Start End Blocks id System
/dev/sdb1 1 654
5253223+ 83 Linux
#主分区
/dev/sdb2 655 2610 15711570
5 Extended
#扩展分区
/dev/sdb5 655 916
2104483+ 83 Linux
#逻辑分区 1
/dev/sdb6 917 1178
2104483+ 83 Linux
#逻辑分区2
Command (m for help): w
#保存并退出
The partition table has been altered!
Calling ioctl。to re-read partition table.
Syncing disks.
[root@localhost -]#
#退回到提示符界面
```

所有的分区立过程中如果不保存并退出是不会生效的，所以建立错了也没有关系，使用 q 命令不保存退出即可。如果使用了 w 命令，就会保存退出。有时因为系统的分区表正忙，所以需要重新启动系统才能使新的分区表生效。命令如下：

```shell
Command (m for help): w
#保存并退出
The partition table has been altered!
Calling ioctl() to re-read partition table.
WARNING: Re-reading the partition table failed with error 16:
Device or resource busy.
The kernel still uses the old table.
The new table will be used at the next reboot.
#要求重新启动，才能格式化
Syncing disks.
```

看到了吗？必须重新启动！可是重新启动很浪费时间。如果不想重新启动，则可以使用 partprobe 命令。这个命令的作用是让系统内核重新读取分区表信息，这样就可以不用重新启动了。命令如下：

```shell
[root@localhost ~]# partprobe
```

如果这个命令不存在，则请安装 parted-2.1-18.el6.i686 这个软件包。partprobe 命令不是必需的，如果没有提示重启系统，则直接格式化即可。

## 15、parted命令用法详解：创建分区

用法：parted [选项]... [设备 [命令 [参数]...]...] 
将带有“参数”的命令应用于“设备”。如果没有给出“命令”，则以交互模式运行. 

帮助选项：

```
-h, --help                    显示此求助信息 
-l, --list                    列出所有设别的分区信息
-i, --interactive             在必要时，提示用户 
-s, --script                  从不提示用户 
-v, --version                 显示版本
```

操作命令：

```shell
cp [FROM-DEVICE] FROM-MINOR TO-MINOR           #将文件系统复制到另一个分区 
help [COMMAND]                                 #打印通用求助信息，或关于 COMMAND 的信息 
mklabel 标签类型                               #创建新的磁盘标签 (分区表) 
mkfs MINOR 文件系统类型                        #在 MINOR 创建类型为“文件系统类型”的文件系统 
mkpart 分区类型 [文件系统类型] 起始点 终止点   #创建一个分区 
mkpartfs 分区类型 文件系统类型 起始点 终止点   #创建一个带有文件系统的分区 
move MINOR 起始点 终止点                       #移动编号为 MINOR 的分区 
name MINOR 名称                                #将编号为 MINOR 的分区命名为“名称” 
print [MINOR]                                  #打印分区表，或者分区 
quit                                           #退出程序 
rescue 起始点 终止点                           #挽救临近“起始点”、“终止点”的遗失的分区 
resize MINOR 起始点 终止点                     #改变位于编号为 MINOR 的分区中文件系统的大小 
rm MINOR                                       #删除编号为 MINOR 的分区 
select 设备                                    #选择要编辑的设备 
set MINOR 标志 状态                            #改变编号为 MINOR 的分区的标志
```

**操作实例：**

a、选择分区硬盘

首先类似fdisk一样，先选择要分区的硬盘，此处为/dev/hdd： ((parted)表示在parted中输入的命令，其他为自动打印的信息)

```shell
[root@10.10.90.97 ~]# parted /dev/hdd
GNU Parted 1.8.1
Using /dev/hdd
Welcome to GNU Parted! Type 'help' to view a list of commands.
```

b、创建分区

选择了/dev/hdd作为我们操作的磁盘，接下来需要创建一个分区表(在parted中可以使用help命令打印帮助信息)：

```
(parted) mklabel
New disk label type? gpt    (我们要正确分区大于2TB的磁盘，应该使用gpt方式的分区表，输入gpt后回车)
```

c、完成分区操作

创建好分区表以后，接下来就可以进行分区操作了，执行mkpart命令，分别输入分区名称，文件系统和分区 的起止位置

```
(parted) mkpart
Partition name? []? dp1
File system type? [ext2]? ext3
Start? 0           （可以用百分比表示，比如Start? 0% , End? 50%）
End? 500GB
```

d、验证分区信息

分好区后可以使用print命令打印分区信息，下面是一个print的样例

```
(parted) print
Model: VBOX HARDDISK (ide)
Disk /dev/hdd: 2199GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Number Start End Size File system Name Flags
1 17.4kB 500GB 500GB dp1
```

e、删除分区示例

如果分区错了，可以使用rm命令删除分区，比如我们要删除上面的分区，然后打印删除后的结果

```
(parted)rm 1               #rm后面使用分区的号码，就是用print打印出来的Number
(parted) print
Model: VBOX HARDDISK (ide)
Disk /dev/hdd: 2199GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Number Start End Size File system Name Flags
```

f、完整示例

按照上面的方法把整个硬盘都分好区，下面是一个分完后的样例

```shell
(parted) mkpart
Partition name? []? dp1
File system type? [ext2]? ext3
Start? 0
End? 500GB
(parted) mkpart
Partition name? []? dp2
File system type? [ext2]? ext3
Start? 500GB
End? 2199GB
(parted) print
Model: VBOX HARDDISK (ide)
Disk /dev/hdd: 2199GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Number Start End Size File system Name Flags
1 17.4kB 500GB 500GB dp1
2 500GB 2199GB 1699GB dp2
```

g、格式化操作

完成以后我们可以使用quit命令退出parted并使用系统的mkfs命令对分区进行格式化了。

```shell
[root@10.10.90.97 ~]# fdisk -l
WARNING: GPT (GUID Partition Table) detected on '/dev/hdd'! The util fdisk doesn't support GPT. Use GNU Parted.
Disk /dev/hdd: 2199.0 GB, 2199022206976 bytes
255 heads, 63 sectors/track, 267349 cylinders
Units = cylinders of 16065 * 512 = 8225280 bytes
Device Boot Start End Blocks Id System
/dev/hdd1 1 267350 2147482623+ ee EFI GPT
[root@10.10.90.97 ~]# mkfs.ext3 /dev/hdd1
[root@10.10.90.97 ~]# mkfs.ext3 /dev/hdd2
[root@10.10.90.97 ~]# mkdir /dp1 /dp2
[root@10.10.90.97 ~]# mount /dev/hdd1 /dp1
[root@10.10.90.97 ~]# mount /dev/hdd2 /dp2
```

## 16、mkfs命令详解:格式化分区（为分区写入文件系统）

分区完成后，如果不格式化写入文件系统，则是不能正常使用的。这时就需要使用 mkfs 命令对硬盘分区进行格式化。

mkfs 命令格式如下：

```
[root@localhost ~]# mkfs [-t 文件系统格式] 分区设备文件名
```

-t 文件系统格式：用于指定格式化的文件系统，如 ext3、ext4；

虽然 mkfs 命令非常简单易用，但其不能调整分区的默认参数（比如块大小是 4096 Bytes），这些默认参数除非特殊清况，否则不需要调整。如果想要调整，就需要使用 mke2fs 命令重新格式化

## 17、mke2fs命令:格式化硬盘（给硬盘写入文件系统）

mke2fs 命令的基本格式如下：

```
[root@localhost ~]# mke2fs [选项] 分区设备文件名
```

表 1 罗列出了 mke2fs 命令常用的几个选项及各自的功能。

| 选项        | 功能                                                    |
| ----------- | ------------------------------------------------------- |
| -t 文件系统 | 指定格式化成哪个文件系统， 如 ext2、ext3、ext4；        |
| -b 字节     | 指定 block 的大小；                                     |
| -i 字节     | 指定"字节 inode "的比例，也就是多少字节分配一个 inode； |
| -j          | 建立带有 ext3 日志功能的文件系统；                      |
| -L 卷标名   | 给文件系统设置卷标名，就不使用 e2label 命令设定了；     |

如果没有特殊需要，建议使用 mkfs 命令对硬盘分区进行格式化。

## 18、Linux虚拟内存和物理内存

我们都知道，直接从内存读写数据要比从硬盘读写数据快得多，因此更希望所有数据的读取和写入都在内存中完成，然而内存是有限的，这样就引出了物理内存与虚拟内存的概念。

物理内存就是系统硬件提供的内存大小，是真正的内存。相对于物理内存，在 Linux 下还有一个虚拟内存的概念，虚拟内存是为了满足物理内存的不足而提出的策略，它是利用磁盘空间虚拟出的一块逻辑内存。用作虚拟内存的磁盘空间被称为交换空间（又称 swap 空间）。

作为物理内存的扩展，Linux 会在物理内存不足时，使用交换分区的虚拟内存，更详细地说，就是内核会将暂时不用的内存块信息写到交换空间，这样一来，物理内存得到了释放，这块内存就可以用于其他目的，当需要用到原始的内容时，这些信息会被重新从交换空间读入物理内存。

Linux 的内存管理采取的是分页存取机制，为了保证物理内存能得到充分的利用，内核会在适当的时候将物理内存中不经常使用的数据块自动交换到虚拟内存中，而将经常使用的信息保留到物理内存。

要深入了解 Linux 内存运行机制，需要知道下面提到的几个方面：

- 首先，Linux 系统会不时地进行页面交换操作，以保持尽可能多的空闲物理内存，即使并没有什么事情需要内存，Linux 也会交换出暂时不用的内存页面，因为这样可以大大节省等待交换所需的时间。
- 其次，Linux 进行页面交换是有条件的，不是所有页面在不用时都交换到虚拟内存，Linux 内核根据“最近最经常使用”算法，仅仅将一些不经常使用的页面文件交换到虚拟内存。


有时我们会看到这么一个现象，Linux 物理内存还有很多，但是交换空间也使用了很多，其实这并不奇怪。例如，一个占用很大内存的进程运行时，需要耗费很多内存资源，此时就会有一些不常用页面文件被交换到虚拟内存中，但后来这个占用很多内存资源的进程结束并释放了很多内存时，刚才被交换出去的页面文件并不会自动交换进物理内存（除非有这个必要），那么此时系统物理内存就会空闲很多，同时交换空间也在被使用，就出现了刚才所说的现象了。

最后，交换空间的页面在使用时会首先被交换到物理内存，如果此时没有足够的物理内存来容纳这些页面，它们又会被马上交换出去，如此一来，虚拟内存中可能没有足够的空间来存储这些交换页面，最终会导致 Linux 出现假死机、服务异常等问题。Linux 虽然可以在一段时间内自行恢复，但是恢复后的系统己经基本不可用了。

因此，合理规划和设计 Linux 内存的使用是非常重要的，关于物理内存和交换空间的大小设置问题，取决于实际所用的硬盘大小，但大致遵循这样一个基本原则：

1. 如果内存较小（根据经验，物理内存小于 4GB），一般设置 swap 分区大小为内存的 2 倍；
2. 如果物理内存大于 4GB，而小于 16GB，可以设置 swap 分区大小等于物理内存；
3. 如果内存大小在 16GB 以上，可以设置 swap 为 0，但并不建议这么做，因为设置一定大小的 swap 分区是有一定作用的。

# Linux高级文件系统管理

本章我们将学习高级文件系统管理，主要包括磁盘配额、 LVM (逻辑卷管理）和RAID (磁盘阵列）。其中：

1. 磁盘配额用来限制普通用户在分区中可以使用的容量和文件个数；
2. LVM 可以在不停机和不损失数据的情况下修改分区大小；
3. RAID 由几块硬盘或分区组成，拥有数据冗余功能，当其中的某块硬盘或分区损坏时，硬盘或分区中保存的数据不会丟失。

## 1、磁盘配额是什么，磁盘配额概述

磁盘配额（Quota）就是 Linux 系统中用来限制特定的普通用户或用户组在指定的分区上占用的磁盘空间或文件个数的。

在此概念中，有以下几个重点需要注意：

1. 磁盘配额限制的用户和用户组，只能是普通用户和用户组，也就是说超级用户 root 是不能做磁盘配额的；
2. 磁盘配额限制只能针对分区，而不能针对某个目录，换句话说，磁盘配额仅能针对文件系统进行限制，举个例子，如果你的 /dev/sda5 是挂载在 /home 底下，那么，在 /home 下的所有目录都会受到磁盘配额的限制；
3. 我们可以限制用户占用的磁盘容量大小（block），当然也能限制用户允许占用的文件个数（inode）。


磁盘配额在实际生活中其实是很常见的，比如，我们的邮箱不管多大，都是有限制的，而不可能无限制地存储邮件；我们可以上传文件的服务器也是有容量限制的；网页中的个人空间也不可能让我们无限制地使用。

磁盘配额就好像我们出租写字楼，虽然整栋楼的空间非常大，但是酬整栋楼的成本太高。我们可以分开出租，用户如果觉得不够用，则还可以租用更大的空间。不过租用是不能随便进行的，其中有几个规矩必须遵守：

- 我的楼是租给外来用户的（普通用户），可以租给一个人（用户）,也可以租给一家公司（用户 组），但是这栋楼的所有权是我的，所以不能租给我自己（root 用户）；
- 如果要租用，则只能在每层租用一定大小的空间，而不能在一个房间中再划分出子空间来租用（配额只能针对分区，而不能限制某个目录）；
- 租户可以决定在某层租用多大的空间（磁盘容量限制）,也可以在某层租用几个人员名额，这样只有这几个人员才能进入本层（文件个数限制）。


磁盘配额要想正常使用，有以下几个前提条件：

1. 内核必须支持磁盘配额。Centos 6.x 以上的 Linux 默认支持磁盘配额，不需要做任何修改。如果不放心，则可以查看内核配置文件，看是否支持磁盘配额。命令如下：

   ```shell
   [root@Test-BK-Dev dev]# grep CONFIG_QUOTA /boot/config-3.10.0-1127.19.1.el7.x86_64 
   CONFIG_QUOTA=y
   CONFIG_QUOTA_NETLINK_INTERFACE=y
   # CONFIG_QUOTA_DEBUG is not set
   CONFIG_QUOTA_TREE=y
   CONFIG_QUOTACTL=y
   CONFIG_QUOTACTL_COMPAT=y
   ```

   可以看到，内核已经支持磁盘配额。如果内核不支持，就需要重新编译内核，加入 quota supper 功能。

2. 系统中必须安装了 Quota 工具。我们的 Linux 中默认安装了 Quoted 工具，查看命令如下：

   ```shell
   [root@Test-BK-Dev dev]#  rpm -qa | grep quota
   quota-nls-4.01-19.el7.noarch
   quota-4.01-19.el7.x86_64
   ```

   要支持磁盘配额的分区必须开启磁盘配额功能。这项功能可以手动开启，不再是默认开启的。

磁盘配额可用于限制每个人可用网页空间、邮件空间以及网络硬盘空间的容量。除此之外，在 Linux 系统资源配置方面，使用磁盘配额，还可以限制某一群组或某一使用者所能使用的最大磁盘配额，以及以 Link 的方式，来使邮件可以作为限制的配额（更改 /var/spool/mail 这个路径）。

### 用户配额和组配额

用户配额是指针对用户个人的配额，而组配额是指针对整个用户组的配额。如果我们需要限制的用户数量并不多，则可以给每个用户单独指定配额。如果用户比较多，那么单独限制太过麻烦，这时我们可以把用户加入某个用户组，然后给组指定配额，就会简单得多。

需要注意的是，组中的用户是共享空间或文件数的。也就是说，如果用户 lamp1、lamp2 和 lamp3 都属于 brother 用户组，我给 brother 用户组分配 100MB 的磁盘空间，那么，这三个用户不是平均分配这 100MB 空间的，而是先到先得，谁先占用，谁就有可能占满这 100MB 空间，后来的就没有空间可用了。

### 磁盘容量限制和文件个数限制

我们除了可以通过限制用户可用的 block 数量来限制用户可用的磁盘容量，也可以通过限制用户可用的 inode 数量来限制用户可以上传或新建的文件个数。

### 软限制和硬限制

软限制可理解为警告限制，硬限制就是真正的限制了。比如，规定软限制为 100MB，硬限制为 200MB,那么，当用户使用的磁盘空间为 100~200MB 时，用户还可以继续上传和新建文件，但是每次登录时都会收到一条警告消息，告诉用户磁盘将满。

### 宽限时间

如果用户的空间占用数处于软限制和硬限制之间，那么系统会在用户登录时警告用户磁盘将满，但是这个警告不会一直进行，而是有时间限制的，这个时间就是宽限时间，默认是 7 天。

如果到达宽限时间，用户的磁盘占用量还超过软限制，那么软限制就会升级为硬限制。也就是说，如果软限制是 100MB，硬限制是 200MB，宽限时间是 7天，此时用户占用了 120MB,那么今后 7 天用户每次登录时都会出现磁盘将满的警告，如果用户置之不理，7 天后这个用户的硬限制就会变成 100MB，而不是 200MB 了。





# Linux系统管理（进程管理、工作管理和系统定时任务）

## 1、Linux进程管理及作用

### A）什么是进程和程序

进程是正在执行的一个程序或命令，每个进程都是一个运行的实体，都有自己的地址空间，并占用一定的系统资源。程序是人使用计算机语言编写的可以实现特定目标或解决特定问题的代码集合。当程序被执行时，执行人的权限和属性，以及程序的代码都会被加载入内存，操作系统给这个进程分配一个 ID，称为 PID（进程 ID）。

也就是说，在操作系统中，所有可以执行的程序与命令都会产生进程。只是有些程序和命令非常简单，如 ls 命令、touch 命令等，它们在执行完后就会结束，相应的进程也就会终结，所以我们很难捕捉到这些进程。但是还有一些程和命令，比如 httpd 进程，启动之后就会一直驻留在系统当中，我们把这样的进程称作常驻内存进程。

某些进程会产生一些新的进程，我们把这些进程称作子进程，而把这个进程本身称作父进程。比如，我们必须正常登录到 Shell 环境中才能执行系统命令，而 Linux 的标准 Shell 是 bash。我们在 bash 当中执行了 ls 命令，那么 bash 就是父进程，而 ls 命令是在 bash 进程中产生的进程，所以 ls 进程是 bash 进程的子进程。也就是说，子进程是依赖父进程而产生的，如果父进程不存在，那么子进程也不存在了

### B）进程管理的作用

那么，进程管理到底应该是做什么的呢？我以为，进程管理主要有以下 3 个作用。

#### 1) 判断服务器的健康状态

运维工程师最主要的工作就是保证服务器安全、稳定地运行。理想的状态是，在服务器出现问题，但是还没有造成服务器宕机或停止服务时，就人为干预解决了问题。

进程管理最主要的工作就是判断服务器当前运行是否健康，是否需要人为干预。如果服务器的 CPU 占用率、内存占用率过高，就需要人为介入解决问题了。这又出现了一个问题，我们发现服务器的 CPU 或内存占用率很高，该如何介入呢？是直接终止高负载的进程吗？

当然不是，应该判断这个进程是否是正常进程，如果是正常进程，则说明你的服务器已经不能满足应用需求，你需要更好的硬件或搭建集群了；如果是非法进程占用了系统资源，则更不能直接中止进程，而要判断非法进程的来源、作用和所在位置，从而把它彻底清除。

当然，如果服务器数量很少，我们完全可以人为通过进程管理命令来进行监控与干预，但如果服务器数量较多，那么人为手工监控就变得非常困难了，这时我们就需要相应的监控服务，如 cacti 或 nagios。总之，进程管理工作中最重要的工作就是判断服务器的健康状 态，最理想的状态是服务器宕机之前就解决问题，从而避免服务器的宕机。

#### 2) 查看系统中所有的进程

我们需要查看看系统中所有正在运行的进程，通过这些进程可以判断系统中运行了哪些服务、是否有非法服务在运行。

#### 3) 杀死进程

这是进程管理中最不常用的手段。当需要停止服务时，会通过正确关闭命令来停止服务（如 apache 服务可以通过 service httpd stop 命令来关闭）。只有在正确终止进程的手段失效的情况下，才会考虑使用 kill 命令杀死进程。

## 2、Linux进程启动的方式有几种？

在 Linux 系统中，每个进程都有一个唯一的进程号（PID），方便系统识别和调度进程。通过简单地输出运行程序的程序名，就可以运行该程序，其实也就是启动了一个进程。

总体来说，启动一个进程主要有 2 种途径，分别是通过手工启动和通过调度启动（事先进行设置，根据用户要求，进程可以自行启动），接下来就一一介绍这 2 中方式。

### A）手工启动进程

手工启动进程指的是由用户输入命令直接启动一个进程，根据所启动的进程类型和性质的不同，其又可以细分为前台启动和后台启动 2 种方式。

#### 前台启动进程

这是手工启动进程最常用的方式，因为当用户输入一个命令并运行，就已经启动了一个进程，而且是一个前台的进程，此时系统其实已经处于一个多进程的状态（一个是 Shell 进程，另一个是新启动的进程）。

#### 后台启动进程

从后台启动进程，其实就是在命令结尾处添加一个 " &" 符号（注意，& 前面有空格）。输入命令并运行之后，Shell 会提供给我们一个数字，此数字就是该进程的进程号。然后直接就会出现提示符，用户就可以继续完成其他工作，例如：

```shell
[root@localhost ~]# find / -name install.log &
[1] 1920
#[1]是工作号，1920是进程号
```

以上介绍了手工启动的 2 种方式，实际上它们有个共同的特点，就是新进程都是由当前 Shell 这个进程产生的，换句话说，是 Shell 创建了新进程，于是称这种关系为进程间的父子关系，其中 Shell 是父进程，新进程是子进程。

值得一提的是，一个父进程可以有多个子进程，通常子进程结束后才能继续父进程；当然，如果是从后台启动，父进程就不用等待子进程了。

### B）Linux调度启动进程

在 Linux 系统中，任务可以被配置在指定的时间、日期或者系统平均负载量低于指定值时自动启动。

例如，Linux 预配置了重要系统任务的运行，以便可以使系统能够实时被更新，系统管理员也可以使用自动化的任务来定期对重要数据进行备份。

## 2、ps命令详解：查看正在运行的进程（Process Status）

ps 命令的基本格式如下：

```shell
[root@localhost ~]# ps aux
\#查看系统中所有的进程，使用 BS 操作系统格式
[root@localhost ~]# ps -le
\#查看系统中所有的进程，使用 Linux 标准命令格式
```

选项：

- a：显示一个终端的所有进程，除会话引线外；
- u：显示进程的归属用户及内存的使用情况；
- x：显示没有控制终端的进程；
- -l：长格式显示更加详细的信息；
- -e：显示所有进程；

可以看到，ps 命令有些与众不同，它的部分选项不能加入"-"，比如命令"ps aux"，其中"aux"是选项，但是前面不能带“-”。

大家如果执行 "man ps" 命令，则会发现 ps 命令的帮助为了适应不同的类 UNIX 系统，可用格式非常多，不方便记忆。所以，我建议大家记忆几个固定选项即可。比如：

- "ps aux" 可以查看系统中所有的进程；
- "ps -le" 可以查看系统中所有的进程，而且还能看到进程的父进程的 PID 和进程优先级；
- "ps -l" 只能看到当前 Shell 产生的进程；

```shell
[root@Test-BK-Dev /]# ps aux
USER        PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root          1  0.0  0.0 193980  7128 ?        Ss   Jul13   0:11 /usr/lib/systemd/systemd --switched-root --system --deserialize 22
root          2  0.0  0.0      0     0 ?        S    Jul13   0:00 [kthreadd]
root          4  0.0  0.0      0     0 ?        S<   Jul13   0:00 [kworker/0:0H]
root          5  0.0  0.0      0     0 ?        S    Jul13   0:01 [kworker/u480:0]
root          6  0.0  0.0      0     0 ?        S    Jul13   0:00 [ksoftirqd/0]
root          7  0.0  0.0      0     0 ?        S    Jul13   0:00 [migration/0]
root          8  0.0  0.0      0     0 ?        S    Jul13   0:00 [rcu_bh]
root          9  0.0  0.0      0     0 ?        S    Jul13   0:39 [rcu_sched]
root         10  0.0  0.0      0     0 ?        S<   Jul13   0:00 [lru-add-drain]
```

|  表头   | 含义                                                         |
| :-----: | ------------------------------------------------------------ |
|  USER   | 该进程是由哪个用户产生的。                                   |
|   PID   | 进程的 ID。                                                  |
|  %CPU   | 该进程占用 CPU 资源的百分比，占用的百分比越高，进程越耗费资源。 |
|  %MEM   | 该进程占用物理内存的百分比，占用的百分比越高，进程越耗费资源。 |
|   VSZ   | 该进程占用虚拟内存的大小，单位为 KB。                        |
|   RSS   | 该进程占用实际物理内存的大小，单位为 KB。                    |
|   TTY   | 该进程是在哪个终端运行的。其中，tty1 ~ tty7 代表本地控制台终端（可以通过 Alt+F1 ~ F7 快捷键切换不同的终端），tty1~tty6 是本地的字符界面终端，tty7 是图形终端。pts/0 ~ 255 代表虚拟终端，一般是远程连接的终端，第一个远程连接占用 pts/0，第二个远程连接占用 pts/1，依次増长。 |
|  STAT   | 进程状态。常见的状态有以下几种：-D：不可被唤醒的睡眠状态，通常用于 I/O 情况。-R：该进程正在运行。-S：该进程处于睡眠状态，可被唤醒。-T：停止状态，可能是在后台暂停或进程处于除错状态。-W：内存交互状态（从 2.6 内核开始无效）。-X：死掉的进程（应该不会出现）。-Z：僵尸进程。进程已经中止，但是部分程序还在内存当中。-<：高优先级（以下状态在 BSD 格式中出现）。-N：低优先级。-L：被锁入内存。-s：包含子进程。-l：多线程（小写 L）。-+：位于后台。 |
|  START  | 该进程的启动时间。                                           |
|  TIME   | 该进程占用 CPU 的运算时间，注意不是系统时间。                |
| COMMAND | 产生此进程的命令名。                                         |

```shell
[root@Test-BK-Dev /]# ps -le
F S   UID    PID   PPID  C PRI  NI ADDR SZ WCHAN  TTY          TIME CMD
4 S     0      1      0  0  80   0 - 48495 ep_pol ?        00:00:13 systemd
1 S     0      2      0  0  80   0 -     0 kthrea ?        00:00:00 kthreadd
1 S     0      4      2  0  60 -20 -     0 worker ?        00:00:00 kworker/0:0H
1 S     0      5      2  0  80   0 -     0 worker ?        00:00:01 kworker/u480:0
1 S     0      6      2  0  80   0 -     0 smpboo ?        00:00:00 ksoftirqd/0
1 S     0      7      2  0 -40   - -     0 smpboo ?        00:00:00 migration/0
1 S     0      8      2  0  80   0 -     0 rcu_gp ?        00:00:00 rcu_bh
1 S     0      9      2  0  80   0 -     0 rcu_gp ?        00:00:47 rcu_sched
1 S     0     10      2  0  60 -20 -     0 rescue ?        00:00:00 lru-add-drain
5 S     0     11      2  0 -40   - -     0 smpboo ?        00:00:00 watchdog/0
```

| 表头  | 含义                                                         |
| :---: | ------------------------------------------------------------ |
|   F   | 进程标志，说明进程的权限，常见的标志有两个: 1：进程可以被复制，但是不能被执行；4：进程使用超级用户权限； |
|   S   | 进程状态。具体的状态和"psaux"命令中的 STAT 状态一致；        |
|  UID  | 运行此进程的用户的 ID；                                      |
|  PID  | 进程的 ID；                                                  |
| PPID  | 父进程的 ID；                                                |
|   C   | 该进程的 CPU 使用率，单位是百分比；                          |
|  PRI  | 进程的优先级，数值越小，该进程的优先级越高，越早被 CPU 执行； |
|  NI   | 进程的优先级，数值越小，该进程越早被执行；                   |
| ADDR  | 该进程在内存的哪个位置；                                     |
|  SZ   | 该进程占用多大内存；                                         |
| WCHAN | 该进程是否运行。"-"代表正在运行；                            |
|  TTY  | 该进程由哪个终端产生；                                       |
| TIME  | 该进程占用 CPU 的运算时间，注意不是系统时间；                |
|  CMD  | 产生此进程的命令名；                                         |

我们再来说说僵尸进程。僵尸进程的产生一般是由于进程非正常停止或程序编写错误，导致子进程先于父进程结束，而父进程又没有正确地回收子进程，从而造成子进程一直存在于内存当中，这就是僵尸进程。

僵尸进程会对主机的稳定性产生影响，所以，在产生僵尸进程后，一定要对产生僵尸进程的软件进行优化，避免一直产生僵尸进程；对于已经产生的僵尸进程，可以在查找出来之后强制中止。

## 3、top命令详解：持续监听进程运行状态

top 命令的基本格式如下：

```shell
[root@localhost ~]#top [选项]
```

选项：

- -d 秒数：指定 top 命令每隔几秒更新。默认是 3 秒；
- -b：使用批处理模式输出。一般和"-n"选项合用，用于把 top 命令重定向到文件中；
- -n 次数：指定 top 命令执行的次数。一般和"-"选项合用；
- -p 进程PID：仅查看指定 ID 的进程；
- -s：使 top 命令在安全模式中运行，避免在交互模式中出现错误；
- -u 用户名：只监听某个用户的进程；


在 top 命令的显示窗口中，还可以使用如下按键，进行一下交互操作：

- ? 或 h：显示交互模式的帮助；
- P：按照 CPU 的使用率排序，默认就是此选项；
- M：按照内存的使用率排序；
- N：按照 PID 排序；
- T：按照 CPU 的累积运算时间排序，也就是按照 TIME+ 项排序；
- k：按照 PID 给予某个进程一个信号。一般用于中止某个进程，信号 9 是强制中止的信号；
- r：按照 PID 给某个进程重设优先级（Nice）值；
- q：退出 top 命令；

top 命令的输出内容是动态的，默认每隔 3 秒刷新一次。

```shell
[root@localhost ~]# top
top - 12:26:46 up 1 day, 13:32, 2 users, load average: 0.00, 0.00, 0.00
Tasks: 95 total, 1 running, 94 sleeping, 0 stopped, 0 zombie
Cpu(s): 0.1%us, 0.1%sy, 0.0%ni, 99.7%id, 0.1%wa, 0.0%hi, 0.1%si, 0.0%st
Mem: 625344k total, 571504k used, 53840k free, 65800k buffers
Swap: 524280k total, 0k used, 524280k free, 409280k cached
```

命令的输出主要分为两部分：

1. 第一部分是前五行，显示的是整个系统的资源使用状况，我们就是通过这些输出来判断服务器的资源使用状态的；
2. 第二部分从第六行开始，显示的是系统中进程的信息；

我们先来说明第一部分的作用。

- 第一行为任务队列信息，具体内容如表 1 所示。

  | 内 容                         | 说 明                                                        |
  | ----------------------------- | ------------------------------------------------------------ |
  | 12:26:46                      | 系统当前时间                                                 |
  | up 1 day, 13:32               | 系统的运行时间.本机己经运行 1 天 13 小时 32 分钟             |
  | 2 users                       | 当前登录了两个用户                                           |
  | load average: 0.00,0.00，0.00 | 系统在之前 1 分钟、5 分钟、15 分钟的平均负载。如果 CPU 是单核的，则这个数值超过 1 就是高负载：如果 CPU 是四核的，则这个数值超过 4 就是高负载 （这个平均负载完全是依据个人经验来进行判断的，一般认为不应该超过服务器 CPU 的核数） |

- 第二行为进程信息，具体内容如表 2 所示。

  | 内 容           | 说 明                                          |
  | --------------- | ---------------------------------------------- |
  | Tasks: 95 total | 系统中的进程总数                               |
  | 1 running       | 正在运行的进程数                               |
  | 94 sleeping     | 睡眠的进程数                                   |
  | 0 stopped       | 正在停止的进程数                               |
  | 0 zombie        | 僵尸进程数。如果不是 0，则需要手工检查僵尸进程 |

- 第三行为 CPU 信息，具体内容如表 3 所示。

  | 内 容           | 说 明                                                        |
  | --------------- | ------------------------------------------------------------ |
  | Cpu(s): 0.1 %us | 用户模式占用的 CPU 百分比                                    |
  | 0.1%sy          | 系统模式占用的 CPU 百分比                                    |
  | 0.0%ni          | 改变过优先级的用户进程占用的 CPU 百分比                      |
  | 99.7%id         | 空闲 CPU 占用的 CPU 百分比                                   |
  | 0.1%wa          | 等待输入/输出的进程占用的 CPU 百分比                         |
  | 0.0%hi          | 硬中断请求服务占用的 CPU 百分比                              |
  | 0.1%si          | 软中断请求服务占用的 CPU 百分比                              |
  | 0.0%st          | st（steal time）意为虚拟时间百分比，就是当有虚拟机时，虚拟 CPU 等待实际 CPU 的时间百分比 |

- 第四行为物理内存信息，具体内容如表 4 所示。

  | 内 容              | 说 明                                                        |
  | ------------------ | ------------------------------------------------------------ |
  | Mem: 625344k total | 物理内存的总量，单位为KB                                     |
  | 571504k used       | 己经使用的物理内存数量                                       |
  | 53840k&ee          | 空闲的物理内存数量。我们使用的是虚拟机，共分配了 628MB内存，所以只有53MB的空闲内存 |
  | 65800k buffers     | 作为缓冲的内存数量                                           |

- 第五行为交换分区（swap）信息，如表 5 所示。

  | 内 容               | 说 明                        |
  | ------------------- | ---------------------------- |
  | Swap: 524280k total | 交换分区（虚拟内存）的总大小 |
  | Ok used             | 已经使用的交换分区的大小     |
  | 524280k free        | 空闲交换分区的大小           |
  | 409280k cached      | 作为缓存的交换分区的大小     |

我们通过 top 命令的第一部分就可以判断服务器的健康状态。如果 1 分钟、5 分钟、15 分钟的平均负载高于 1，则证明系统压力较大。如果 CPU 的使用率过高或空闲率过低，则证明系统压力较大。如果物理内存的空闲内存过小，则也证明系统压力较大。

这时，我们就应该判断是什么进程占用了系统资源。如果是不必要的进程，就应该结束这些进程；如果是必需进程，那么我们该増加服务器资源（比如増加虚拟机内存），或者建立集群服务器。

我们还要解释一下缓冲（buffer）和缓存（cache）的区别：

- 缓存（cache）是在读取硬盘中的数据时，把最常用的数据保存在内存的缓存区中，再次读取该数据时，就不去硬盘中读取了，而在缓存中读取。
- 缓冲（buffer）是在向硬盘写入数据时，先把数据放入缓冲区,然后再一起向硬盘写入，把分散的写操作集中进行，减少磁盘碎片和硬盘的反复寻道，从而提高系统性能。

简单来说，缓存（cache）是用来加速数据从硬盘中"读取"的，而缓冲（buffer）是用来加速数据"写入"硬盘的。

再来看 top 命令的第二部分输出，主要是系统进程信息，各个字段的含义如下：

- PID：进程的 ID。
- USER：该进程所属的用户。
- PR：优先级，数值越小优先级越高。
- NI：优先级，数值越小、优先级越高。
- VIRT：该进程使用的虚拟内存的大小，单位为 KB。
- RES：该进程使用的物理内存的大小，单位为 KB。
- SHR：共享内存大小，单位为 KB。
- S：进程状态。
- %CPU：该进程占用 CPU 的百分比。
- %MEM：该进程占用内存的百分比。
- TIME+：该进程共占用的 CPU 时间。
- COMMAND：进程的命令名。

如果在操作终端执行 top 命令，则并不能看到系统中所有的进程，默认看到的只是 CPU 占比靠前的进程。如果我们想要看到所有的进程，则可以把 top 命令的执行结果重定向到文件中。不过 top 命令是持续运行的，这时就需要使用 "-b" 和 "-n" 选项了。具体命令如下：

```shell
[root@localhost ~]# top -b -n 1 > /root/top.log
#让top命令只执行一次，然后把执行结果保存到top.log文件中，这样就能看到所有的进程了
```

## 4、pstree命令：查看进程树

pstree 命令是以树形结构显示程序和进程之间的关系，此命令的基本格式如下：

```shell
[root@localhost ~]# pstree [选项] [PID或用户名]
```

pstree命令需要安装：

```shell
yum install psmisc -y 
```

表 1 罗列出了 pstree 命令常用选项以及各自的含义。

| 选项 | 含义                                                         |
| ---- | ------------------------------------------------------------ |
| -a   | 显示启动每个进程对应的完整指令，包括启动进程的路径、参数等。 |
| -c   | 不使用精简法显示进程信息，即显示的进程中包含子进程和父进程。 |
| -n   | 根据进程 PID 号来排序输出，默认是以程序名排序输出的。        |
| -p   | 显示进程的 PID。                                             |
| -u   | 显示进程对应的用户名称。                                     |

需要注意的是，在使用 pstree 命令时，如果不指定进程的 PID 号，也不指定用户名称，则会以 init 进程为根进程，显示系统中所有程序和进程的信息；反之，若指定 PID 号或用户名，则将以 PID 或指定命令为根进程，显示 PID 或用户对应的所有程序和进程。

<!--init 进程是系统启动的第一个进程，进程的 PID 是 1，也是系统中所有进程的父进程。-->

## 5、lsof命令：列出进程调用或打开的文件信息

lsof 命令，“list opened files”的缩写，直译过来，就是列举系统中已经被打开的文件。通过 lsof 命令，我们就可以根据文件找到对应的进程信息，也可以根据进程信息找到进程打开的文件。

lsof 命令的基本格式如下：

```shell
[root@localhost ~]# lsof [选项]
```

此命令常用的选项及功能，如表 1 所示。

| 选项      | 功能                                 |
| --------- | ------------------------------------ |
| -c 字符串 | 只列出以字符串开头的进程打开的文件。 |
| +d 目录名 | 列出某个目录中所有被进程调用的文件。 |
| -u 用户名 | 只列出某个用户的进程打开的文件。     |
| -p pid    | 列出某个 PID 进程打开的文件。        |

```shell
yum install lsof -y
```

## 6、Linux进程优先级

Linux 是一个多用户、多任务的操作系统，系统中通常运行着非常多的进程。但是 CPU 在一个时钟周期内只能运算一条指令（现在的 CPU 采用了多线程、多核心技术，所以在一个时钟周期内可以运算多条指令。 但是同时运算的指令数也远远小于系统中的进程总数），那问题来了：谁应该先运算，谁应该后运算呢？这就需要由进程的优先级来决定了。

另外，CPU 在运算数据时，不是把一个集成算完成，再进行下一个进程的运算，而是先运算进程 1，再运算进程 2，接下来运算进程 3，然后再运算进程 1，直到进程任务结束。不仅如此，由于进程优先级的存在，进程并不是依次运算的，而是哪个进程的优先级高，哪个进程会在一次运算循环中被更多次地运算。

这样说很难理解，我们换一种说法。假设我现在有 4 个孩子（进程）需要喂饭（运算），我更喜欢孩子 1（进程 1 优先级更高），孩子 2、孩子 3 和孩子 4 一视同仁（进程 2、进程 3 和进程 4 的优先级一致）。现在我开始喂饭了，我不能先把孩子 1 喂饱，再喂其他的孩子，而是需要循环喂饭（CPU 运算时所有进程循环运算）。那么，我在喂饭时（运算），会先喂孩子 1 一口饭，然后再去喂其他孩子。而且在一次循环中，先喂孩子 1 两口饭，因为我更喜欢孩子 1（优先级高），而喂其他的孩子一口饭。这样，孩子 1 会先吃饱（进程 1 运算得更快），因为我更喜欢孩子 1。

在 Linux 系统中，表示进程优先级的有两个参数：Priority 和 Nice。还记得 "ps -le" 命令吗？

```shell
[root@localhost ~]# ps -le
F S UID PID PPID C PRI NI ADDR  SZ WCHAN TTY    TIME  CMD
4 S   0   1    0 0  80  0    - 718     -   ? 00:00:01 init
1 S   0   2    0 0  80  0    -   0     -   ? 00:00:00 kthreadd
...省略部分输出... 
```

其中，PRI 代表 Priority，NI 代表 Nice。这两个值都表示优先级，数值越小代表该进程越优先被 CPU 处理。不过，PRI值是由内核动态调整的，用户不能直接修改。所以我们只能通过修改 NI 值来影响 PRI 值，间接地调整进程优先级。

PRI 和 NI 的关系如下：

PRI (最终值) = PRI (原始值) + NI

其实，大家只需要记得，我们修改 NI 的值就可以改变进程的优先级即可。NI 值越小，进程的 PRI 就会降低，该进程就越优先被 CPU 处理；反之，NI 值越大，进程的 PRI 值就会増加，该进程就越靠后被 CPU 处理。

修改 NI 值时有几个注意事项：

- NI 范围是 -20~19。
- 普通用户调整 NI 值的范围是 0~19，而且只能调整自己的进程。
- 普通用户只能调高 NI 值，而不能降低。如原本 NI 值为 0，则只能调整为大于 0。
- 只有 root 用户才能设定进程 NI 值为负值，而且可以调整任何用户的进程。

## 7、nice和renice命令：改变进程优先级

当 Linux 内核尝试决定哪些运行中的进程可以访问 CPU 时，其中一个需要考虑的因素就是进程优先级的值（也称为 nice 值）。每个进程都有一个介于 -20 到 19 之间的 nice 值。默认情况下，进程的 nice 值为 0。

进程的 nice 值，可以通过 nice 命令和 renice 命令修改，进而调整进程的运行顺序。

### nice命令

nice 命令可以给要启动的进程赋予 NI 值，但是不能修改已运行进程的 NI 值。

nice 命令格式如下：

```shell
[root@localhost ~] # nice [-n NI值] 命令
```

-n NI值：给命令赋予 NI 值，该值的范围为 -20~19；

```shell
[root@localhost ~]# service httpd start
[root@localhost ~]# ps -le 丨 grep "httd" | grep -v grep
F S UID  PID PPID C PRI NI ADDR   SZ   WCHAN TTY      TIME   CMD
1 S   0 2084    1 0 80   0    - 1130     -     ?  00:00:00 httpd
5 S   2 2085 2084 0 80   0    - 1130     -     ?  00:00:00 httpd
5 S   2 2086 2084 0 80   0    - 1130     -     ?  00:00:00 httpd
5 S   2 2087 2084 0 80   0    - 1130     -     ?  00:00:00 httpd
5 S   2 2088 2084 0 80   0    - 1130     -     ?  00:00:00 httpd
5 S   2 2089 2084 0 80   0    - 1130     -     ?  00:00:00 httpd
#用默认优先级自动apache服务，PRI值是80，而NI值是0
[root@localhost ~]# service httpd stop
#停止apache服务
[root@localhost ~]# nice -n -5 service httpd start
#启动apache服务，同时修改apache服务进程的NI值为-5
[rooteiocdlhost ~]# ps -le | grep "httpd" | grep -v grep
F S UID  PID PPID C FRI NI ADDR    SZ WCHAN TTY      TIME   CMD
1 S   0 2122    1 0 75   5    -  1130    -    ?  00:00:00 httpd
5 S   2 2123 2122 0 75   5    -  1130    -    ?  00:00:00 httpd
5 S   2 2124 2122 0 75   5    -  1130    -    ?  00:00:00 httpd
5 S   2 2125 2122 0 75   5    -  1130    -    ?  00:00:00 httpd
5 S   2 2126 2122 0 75   5    -  1130    -    ?  00:00:00 httpd
5 S   2 2127 2122 0 75   5    -  1130    -    ?  00:00:00 httpd
#httpd进程的PRI值变为了75，而NI值为-5
```

### renice 命令

同 nice 命令恰恰相反，renice 命令可以在进程运行时修改其 NI 值，从而调整优先级。

renice 命令格式如下：

```shell
[root@localhost ~] # renice [优先级] PID
```

注意，此命令中使用的是进程的 PID 号，因此常与 ps 等命令配合使用。

```shell
[root@localhost ~]# renice -10 2125
2125: old priority -5, new priority -10
[root@localhost ~]# ps -le | grep "httpd" | grep -v grep
1 S 0 2122 1 0 75 -5 - 113.0 - ? 00:00:00 httpd
5 S 2 2123 2122 0 75 -5 - 1130 - ? 00:00:00 httpd
5 S 2 2124 2122 0 75 -5 - 1130 - ? 00:00:00 httpd
5 S 2 2125 2122 0 70 -10 - 1130 - ? 00:00:00 httpd
5 S 2 2126 2122 0 75 -5 - 1130 - ? 00:00:00 httpd
5 S 2 2.127 2122 0 75 -5 - 1130 - ? 00:00:00 httpd
#PID为2125的进程的PRI值为70，而NI值为-10
```

如何合理地设置进程优先级，曾经是一件让系统管理员非常费神的事情。但现在已经不是了，如何地 CPU 足够强大，能够合理地对进程进行调整，输入输出设备也远远跟不上 CPU 地脚步，反而在更多的情况下，CPU 总是在等待哪些缓慢的 I/O（输入/输出）设备完成数据的读写和传输任务。

然而，手动设置进程的优先级并不能影响 I/O 设备对它的处理，这就意味着，哪些有着低优先级的进程常常不合理地占据着本就低效地 I/O 资源。

## 8、Linux常用信号（进程间通信）及其含义

进程的管理主要是指进程的关闭与重启。我们一般关闭或重启软件，都是关闭或重启它的程序，而不是直接操作进程的。比如，要重启 apache 服务，一般使用命令"service httpd restart"重启 apache的程序。

那么，可以通过直接管理进程来关闭或重启 apache 吗？答案是肯定的，这时就要依赖进程的信号（Signal）了。我们需要给予该进程号，告诉进程我们想要让它做什么。

系统中可以识别的信号较多，我们可以使用命令"kill -l"或"man 7 signal"来查询。命令如下：

```
[root@Test-BK-Dev ~]# kill -l
 1) SIGHUP	 2) SIGINT	 3) SIGQUIT	 4) SIGILL	 5) SIGTRAP
 6) SIGABRT	 7) SIGBUS	 8) SIGFPE	 9) SIGKILL	10) SIGUSR1
11) SIGSEGV	12) SIGUSR2	13) SIGPIPE	14) SIGALRM	15) SIGTERM
16) SIGSTKFLT	17) SIGCHLD	18) SIGCONT	19) SIGSTOP	20) SIGTSTP
21) SIGTTIN	22) SIGTTOU	23) SIGURG	24) SIGXCPU	25) SIGXFSZ
26) SIGVTALRM	27) SIGPROF	28) SIGWINCH	29) SIGIO	30) SIGPWR
31) SIGSYS	34) SIGRTMIN	35) SIGRTMIN+1	36) SIGRTMIN+2	37) SIGRTMIN+3
38) SIGRTMIN+4	39) SIGRTMIN+5	40) SIGRTMIN+6	41) SIGRTMIN+7	42) SIGRTMIN+8
43) SIGRTMIN+9	44) SIGRTMIN+10	45) SIGRTMIN+11	46) SIGRTMIN+12	47) SIGRTMIN+13
48) SIGRTMIN+14	49) SIGRTMIN+15	50) SIGRTMAX-14	51) SIGRTMAX-13	52) SIGRTMAX-12
53) SIGRTMAX-11	54) SIGRTMAX-10	55) SIGRTMAX-9	56) SIGRTMAX-8	57) SIGRTMAX-7
58) SIGRTMAX-6	59) SIGRTMAX-5	60) SIGRTMAX-4	61) SIGRTMAX-3	62) SIGRTMAX-2
63) SIGRTMAX-1	64) SIGRTMAX	
```

这里介绍一下常见的进程信号，如表 1 所示。

| 信号代号 | 信号名称 | 说 明                                                        |
| -------- | -------- | ------------------------------------------------------------ |
| 1        | SIGHUP   | 该信号让进程立即关闭.然后重新读取配置文件之后重启            |
| 2        | SIGINT   | 程序中止信号，用于中止前台进程。相当于输出 Ctrl+C 快捷键     |
| 8        | SIGFPE   | 在发生致命的算术运算错误时发出。不仅包括浮点运算错误，还包括溢出及除数为 0 等其他所有的算术运算错误 |
| 9        | SIGKILL  | 用来立即结束程序的运行。本信号不能被阻塞、处理和忽略。般用于强制中止进程 |
| 14       | SIGALRM  | 时钟定时信号，计算的是实际的时间或时钟时间。alarm 函数使用该信号 |
| 15       | SIGTERM  | 正常结束进程的信号，kill 命令的默认信号。如果进程已经发生了问题，那么这 个信号是无法正常中止进程的，这时我们才会尝试 SIGKILL 信号，也就是信号 9 |
| 18       | SIGCONT  | 该信号可以让暂停的进程恢复执行。本信号不能被阻断             |
| 19       | SIGSTOP  | 该信号可以暂停前台进程，相当于输入 Ctrl+Z 快捷键。本信号不能被阻断 |

我们只介绍了常见的进程信号，其中最重要的就是 "1"、"9"、"15"这三个信号，我们只需要记住这三个信号即可。

关于如何把这些信号传递给进程，从而控制这个进程，这就需要使用 kill、killall 以及 pkill 命令了，我们会在后续章节中详解介绍这 3 个命令。

## 9、kill命令详解：终止进程

kill 从字面来看，就是用来杀死进程的命令，但事实上，这个或多或少带有一定的误导性。从本质上讲，kill 命令只是用来向进程发送一个信号，至于这个信号是什么，是用户指定的。

也就是说，kill 命令的执行原理是这样的，kill 命令会向操作系统内核发送一个信号（多是终止信号）和目标进程的 PID，然后系统内核根据收到的信号类型，对指定进程进行相应的操作。

kill 命令的基本格式如下：

[root@localhost ~]# kill [信号] PID

kill 命令是按照 PID 来确定进程的，所以 kill 命令只能识别 PID，而不能识别进程名。Linux 定义了几十种不同类型的信号，读者可以使用 kill -l 命令查看所有信号及其编号，这里仅列出几个常用的信号，如表 1 所示。

| 信号编号 | 信号名 | 含义                                                         |
| -------- | ------ | ------------------------------------------------------------ |
| 0        | EXIT   | 程序退出时收到该信息。                                       |
| 1        | HUP    | 挂掉电话线或终端连接的挂起信号，这个信号也会造成某些进程在没有终止的情况下重新初始化。 |
| 2        | INT    | 表示结束进程，但并不是强制性的，常用的 "Ctrl+C" 组合键发出就是一个 kill -2 的信号。 |
| 3        | QUIT   | 退出。                                                       |
| 9        | KILL   | 杀死进程，即强制结束进程。                                   |
| 11       | SEGV   | 段错误。                                                     |
| 15       | TERM   | 正常结束进程，是 kill 命令的默认信号。                       |

需要注意的是，表中省略了各个信号名称的前缀 SIG，也就是说，SIGTERM 和 TERM 这两种写法都对，kill 命令都可以理解。

【例 1】 标准 kill 命令。

```shell
[root@localhost ~】# service httpd start
#启动RPM包默认安装的apache服务
[root@localhost ~]# pstree -p 丨 grep httpd | grep -v "grep"
#查看 httpd 的进程树及 PID。grep 命令査看 httpd 也会生成包含"httpd"关键字的进程，所以使用“-v”反向选择包含“grep”关键字的进程，这里使用 pstree 命令来查询进程，当然也可以使用 ps 和 top 命令
|-httpd(2246)-+-httpd(2247)
|  |-httpd(2248)
|  |-httpd(2249)
|  |-httpd(2250)
|  |-httpd(2251)
[root@localhost ~]# kill 2248
#杀死PID是2248的httpd进程，默认信号是15，正常停止
#如果默认信号15不能杀死进程，则可以尝试-9信号，强制杀死进程
[root@localhost ~]# pstree -p | grep httpd | grep -v "grep"
|-httpd(2246>-+-httpd(2247)
|  |-httpd(2249)
|  |-httpd(2250)
|  |-httpd(2251)
#PID是2248的httpd进程消失了
```


【例 2】使用“-1”信号，让进程重启。

```shell
[root@localhost ~]# kill -1 2246
#使用“-1 (数字1)”信号，让httpd的主进程重新启动
[root@localhost ~]# pstree -p | grep httpd | grep -v "grep"
|-httpd(2246)-+-httpd(2270)
|  |-httpd(2271)
|  |-httpd(2272)
|  |-httpd(2273)
|  |-httpd(2274)
#子httpd进程的PID都更换了，说明httpd进程已经重启了一次
```


【例 3】 使用“-19”信号，让进程暂停。

```shell
[root@localhost ~]# vi test.sh #使用[vi命令](http://c.biancheng.net/vi/)编辑一个文件，不要退出
[root@localhost ~]# ps aux | grep "vi" | grep -v "grep"
root 2313 0.0 0.2 7116 1544 pts/1 S+ 19:2.0 0:00 vi test.sh
#换一个不同的终端，查看一下这个进程的状态。进程状态是S（休眠）和+（位于后台），因为是在另一个终端运行的命令
[root@localhost ~]# kill -19 2313
#使用-19信号，让PID为2313的进程暂停。相当于在vi界面按 Ctrl+Z 快捷键
[root@localhost ~]# ps aux | grep "vi" | grep -v "grep"
root 2313 0.0 0.2 7116 1580 pts/1 T 19:20 0:00 vi test.sh
#注意2313进程的状态，变成了 T（暂停）状态。这时切换回vi的终端,发现vi命令已经暂停，又回到了命令提示符，不过2313进程就会卡在后台。如果想要恢复，可以使用"kill -9 2313”命令强制中止进程，也可以利用后续章节将要学习的工作管理来进行恢复
```

学会如何使用 kill 命令之后，再思考一个问题，使用 kill 命令一定可以终止一个进程吗？

答案是否定的。文章开头说过，kill 命令只是“发送”一个信号，因此，只有当信号被程序成功“捕获”，系统才会执行 kill 命令指定的操作；反之，如果信号被“封锁”或者“忽略”，则 kill 命令将会失效。

## 10、killall命令：终止特定的一类进程

killall 也是用于关闭进程的一个命令，但和 kill 不同的是，killall 命令不再依靠 PID 来杀死单个进程，而是通过程序的进程名来杀死一类进程，也正是由于这一点，该命令常与 ps、pstree 等命令配合使用。

killall 命令的基本格式如下：

```shell
[root@localhost ~]# killall [选项] [信号] 进程名
```

注意，此命令的信号类型同 kill 命令一样，因此这里不再赘述，此命令常用的选项有如下 2 个：

- -i：交互式，询问是否要杀死某个进程；
- -I：忽略进程名的大小写；

## 11、pkill命令：终止进程，按终端号踢出用户

当作于管理进程时，pkill 命令和 killall 命令的用法相同，都是通过进程名杀死一类进程，该命令的基本格式如下：

```shell
[root@localhost ~]# pkill [信号] 进程名
```

下表罗列了此命令常用的信号及其含义。

| 信号编号 | 信号名 | 含义                                                         |
| -------- | ------ | ------------------------------------------------------------ |
| 0        | EXIT   | 程序退出时收到该信息。                                       |
| 1        | HUP    | 挂掉电话线或终端连接的挂起信号，这个信号也会造成某些进程在没有终止的情况下重新初始化。 |
| 2        | INT    | 表示结束进程，但并不是强制性的，常用的 "Ctrl+C" 组合键发出就是一个 kill -2 的信号。 |
| 3        | QUIT   | 退出。                                                       |
| 9        | KILL   | 杀死进程，即强制结束进程。                                   |
| 11       | SEGV   | 段错误。                                                     |
| 15       | TERM   | 正常结束进程，是 kill 命令的默认信号。                       |

除此之外，pkill 还有一个更重要的功能，即按照终端号来踢出用户登录，此时的 pkill 命令的基本格式如下：

```shell
[root@localhost ~]# pkill [-t 终端号] 进程名
```

[-t 终端号] 选项用于按照终端号踢出用户；

学习 killall 命令时，不知道大家发现没有，通过 killall 命令杀死 sshd 进程的方式来踢出用户，非常容易误杀死进程，要么会把 sshd 服务杀死，要么会把自己的登录终端杀死。

所以，不管是使用 kill 命令按照 PID 杀死登录进程，还是使用 killall 命令按照进程名杀死登录进程，都是非常容易误杀死进程的，而使用 pkill 命令则不会，举个例子：

```shell
[root@Test-BK-Dev /]# w
#使用w命令查询本机已经登录的用户
 09:24:32 up 1 day, 22:50,  2 users,  load average: 0.00, 0.01, 0.05
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
root     pts/0    10.6.115.133     Wed09    0.00s  0.93s  0.00s w
root     pts/1    10.6.115.133     Wed09   16:30m  0.03s  0.03s -bash
[root@Test-BK-Dev /]# pkill -9 -t pts/1
#强制杀死从pts/1虚拟终端登陆的进程
[root@Test-BK-Dev /]# w
 09:26:14 up 1 day, 22:51,  1 user,  load average: 0.00, 0.01, 0.05
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
root     pts/0    10.6.115.133     Wed09    6.00s  0.94s  0.00s w
```

## 12、Linux工作管理简介

工作管理指的是在单个登录终端（也就是登录的 Shell 界面）同时管理多个工作的行为。也就是说，我们登陆了一个终端，已经在执行一个操作，那么是否可以在不关闭当前操作的情况下执行其他操作呢？

当然可以，我们可以再启动一个终端，然后执行其他的操作。不过，是否可以在一个终端执行不同的操作呢？这就需要通过工作管理来实现了。

例如，我在当前终端正在 vi 一个文件，在不停止 vi 的情况下，如果我想在同一个终端执行其他的命令，就应该把 vi 命令放入后台，然后再执行其他命令。把命令放入后台，然后把命令恢复到前台，或者让命令恢复到后台执行，这些管理操作就是工作管理。

后台管理有几个事项需要大家注意：

1. 前台是指当前可以操控和执行命令的这个操作环境；后台是指工作可以自行运行，但是不能直接用 Ctrl+C 快捷键来中止它，只能使用 fg/bg 来调用工作。
2. 当前的登录终端只能管理当前终端的工作，而不能管理其他登录终端的工作。比如 tty1 登录的终端是不能管理 tty2 终端中的工作的。
3. 放入后台的命令必须可以持续运行一段时间，这样我们才能捕捉和操作它。
4. 放入后台执行的命令不能和前台用户有交互或需要前台输入，否则只能放入后台暂停，而不能执行。比如 vi 命令只能放入后台暂停，而不能执行，因为 vi 命令需要前台输入信息；top 命令也不能放入后台执行，而只能放入后台暂停，因为 top 命令需要和前台交互。

## 13、Linux命令放入后台运行方法（&和Ctrl+Z）

Linux 命令放入后台的方法有两种，分别介绍如下。

"命令 &"，把命令放入后台执行

第一种把命令放入后台的方法是在命令后面加入 `空格 &`。使用这种方法放入后台的命令，在后台处于执行状态。

```shell
[root@localhost ~]#find / -name install.log &
[1] 1920
#[工作号] 进程号
#把find命令放入后台执行，每个后台命令会被分配一个工作号。命令既然可以执行，就会有进程产生，所以也会有进程号
```

这样，虽然 find 命令在执行，但在当前终端仍然可以执行其他操作。如果在终端上出现如下信息：

```shell
[1]+ Done find / -name install.log
```

则证明后台的这个命令已经完成了。当然，命令如果有执行结果，则也会显示到操作终端上。其中，[1] 是这个命令的工作号，"+"代表这个命令是最近一个被放入后台的。

命令执行过裎中按 Ctrl+Z 快捷键，命令在后台处于暂停状态。

使用这种方法放入后台的命令，就算不和前台有交互，能在后台执行，也处于暂停状态，因为 Ctrl+Z 快捷键就是暂停的快捷键。

每个被放入后台的命令都会被分配一个工作号。第一个被放入后台的命令，工作号是 1；第二个被放入后台的命令，工作号是 2，以此类推。

## 14、 jobs命令：查看当前终端放入后台的工作

jobs 命令的基本格式如下：

```shell
[root@localhost ~]#jobs [选项]
```

表 1 罗列了 jobs 命令常用的选项及含义。

| 选项           | 含义                                   |
| -------------- | -------------------------------------- |
| -l（L 的小写） | 列出进程的 PID 号。                    |
| -n             | 只列出上次发出通知后改变了状态的进程。 |
| -p             | 只列出进程的 PID 号。                  |
| -r             | 只列出运行中的进程。                   |
| -s             | 只列出已停止的进程。                   |

例如：

```shell
[root@localhost ~]#jobs -l
[1]- 2023 Stopped top
[2]+ 2034 Stopped tar -zcf etc.tar.gz /etc
```

可以看到，当前终端有两个后台工作：一个是 top 命令，工作号为 1，状态是暂停，标志是"-"；另一个是 tar 命令，工作号为 2，状态是暂停，标志是"+"。"+"号代表最近一个放入后台的工作，也是工作恢复时默认恢复的工作。"-"号代表倒数第二个放入后台的工作，而第三个以后的工作就没有"+-"标志了。

一旦当前的默认工作处理完成，则带减号的工作就会自动成为新的默认工作，换句话说，不管此时有多少正在运行的工作，任何时间都会有且仅有一个带加号的工作和一个带减号的工作。

##  15、fg命令：把后台命令恢复在前台执行

fg 命令用于把后台工作恢复到前台执行，该命令的基本格式如下：

```shell
[root@localhost ~]#fg %工作号
```

<!--注意，在使用此命令时，％ 可以省略，但若将`% 工作号`全部省略，则此命令会将带有 + 号的工作恢复到前台-->

top 命令是不能在后台执行的，所以，如果要想中止 top 命令，要么把 top 命令恢复到前台，然后正常退出；要么找到 top 命令的 PID，使用 kill 命令杀死这个进程。

## 16、bg命令：把后台暂停的工作恢复到后台执行

bg 命令的基本格式如下：

```shell
[root@localhost ~]# bg ％工作号
```

和 fg 命令类似，这里的 % 可以省略。

举个例子，读者可以试着把前面章节中放入后台的两个工作恢复运行，命令如下：

```shell
[root@localhost ~]# bg ％1  <--- 等同于 bg 1
[root@localhost ~]# bg ％2  <--- 等同于 bg 2
\#把两个命令恢复到后台执行
[root@localhost @]# jobs
[1]+ Stopped top
[2]- Running tar -zcf etc.tar.gz /etc &
\#tar命令的状态变为了Running，但是top命令的状态还是Stopped
```

可以看到，tar 命令确实已经在后台执行了，但是 top 命令怎么还处于暂停状态呢？原因很简单，top 命令是需要和前台交互的，所以不能在后台执行。换句话说，top 命令就是给前台用户显示系统性能的命令，如果 top 命令在后台恢复运行了，那么给谁去看结果呢？

## 17、nohup命令：后台命令脱离终端运行

在前面章节中，我们一直在说进程可以放到后台运行，这里的后台，其实指的是当前登陆终端的后台。这种情况下，当我们以远程管理服务器的方式，在远程终端执行后台命令，如果在命令尚未执行完毕时就退出登陆，那么这个后台命令还会继续执行吗？

当然不会，此命令的执行会被中断。这就引出一个问题，如果我们确实需要在远程终端执行某些后台命令，该如何执行呢？有以下 3 种方法：

1. 把需要在后台执行的命令加入 /etc/rc.local 文件，让系统在启动时执行这个后台程序。这种方法的问题是，服务器是不能随便重启的，如果有临时后台任务，就不能执行了。
2. 使用系统定时任务，让系统在指定的时间执行某个后台命令。这样放入后台的命令与终端无关，是不依赖登录终端的。
3. 使用 nohup 命令。

nohup 命令的作用就是让后台工作在离开操作终端时，也能够正确地在后台执行。此命令的基本格式如下：

```shell
[root@localhost ~]# nohup [命令] &
```

注意，这里的‘&’表示此命令会在终端后台工作；反之，如果没有‘&’，则表示此命令会在终端前台工作。

例如：

```shell
[root@localhost ~]# nohup find / -print > /root/file.log &
[3] 2349
#使用find命令，打印/下的所有文件。放入后台执行
[root@localhost ~]# nohup：忽略输入并把输出追加到"nohup.out"
[root@localhost ~]# nohup：忽略输入并把输出追加到"nohup.out"
#有提示信息
```

接下来的操作要迅速，否则 find 命令就会执行结束。然后我们可以退出登录，重新登录之后，执行“ps aux”命令，会发现 find 命令还在运行。

如果 find 命令执行太快，我们就可以写一个循环脚本，然后使用 nohup 命令执行。例如：

```shell
[root@localhost ~]# vi for.sh
#！/bin/bash
for ((i=0;i<=1000;i=i+1))
#循环1000次
do
echo 11 >> /root/for.log
#在for.log文件中写入11
sleep 10s
#每次循环睡眠10秒
done
[root@localhost ~]# chmod 755 for.sh
[root@localhost ~]# nohup /root/for.sh &
[1] 2478
[root@localhost ~]# nohup：忽略输入并把输出追加到"nohup.out"
#执行脚本
```

接下来退出登录，重新登录之后，这个脚本仍然可以通过“ps aux”命令看到。

## 18、at命令详解：定时执行任务

at：一次性定时任务计划执行

```
yum -y install at
service atd start
chkconfig atd on
```

访问控制指的是允许哪些用户使用 at 命令设定定时任务，或者不允许哪些用户使用 at 命令。大家可以将其想象成设定黑名单或白名单，这样更容易理解。

at 命令的访问控制是依靠 /etc/at.allow（白名单）和 /etc/at.deny（黑名单）这两个文件来实现的，具体规则如下：

- 如果系统中有 /etc/at.allow 文件，那么只有写入 /etc/at.allow 文件（白名单）中的用户可以使用 at 命令，其他用户不能使用 at 命令（注意，/etc/at.allow 文件的优先级更高，也就是说，如果同一个用户既写入 /etc/at.allow 文件，又写入 /etc/at.deny 文件，那么这个用户是可以使用 at 命令的）。
- 如果系统中没有 /etc/at.allow 文件，只有 /etc/at.deny 文件，那么写入 /etc/at.deny 文件（黑名单）中的用户不能使用 at 命令，其他用户可以使用 at 命令。不过这个文件对 root 用户不生效。
- 如果系统中这两个文件都不存在，那么只有 root 用户可以使用 at 命令。

系统中默认只有 /etc/at.deny 文件，而且这个文件是空的，因此，系统中所有的用户都可以使用 at 命令。不过，如果我们打算控制用户的 at 命令权限，那么只需把用户写入 /etc/at.deny 文件即可。

at 命令的格式非常简单，基本格式如下：

```shell
[root@localhost ~] # at [选项] [时间]
```

有关此命令常用的几个选项及各自含义如表 1 所示。

| 选项          | 含义                                                         |
| ------------- | ------------------------------------------------------------ |
| -m            | 当 at 工作完成后，无论命令是否输出，都用 E-mail 通知执行 at 命令的用户。 |
| -c 工作标识号 | 显示该 at 工作的实际内容。                                   |
| -t 时间       | 在指定时间提交工作并执行，时间格式为 [[CC]YY]MMDDhhmm。      |
| -d            | 删除某个工作，需要提供相应的工作标识号（ID），同 atrm 命令的作用相同。 |
| -l            | 列出当前所有等待运行的工作，和 atq 命令具有相同的额作用。    |
| -f 脚本文件   | 指定所要提交的脚本文件。                                     |

另外，下表 罗列了此命令中关于时间参数可用的以下格式

| 格式                       | 用法                                                         |
| -------------------------- | ------------------------------------------------------------ |
| HH:MM                      | 比如 04:00 AM。如果时间已过，则它会在第二天的同一时间执行。  |
| Midnight（midnight）       | 代表 12:00 AM（也就是 00:00）。                              |
| Noon（noon）               | 代表 12:00 PM（相当于 12:00）。                              |
| Teatime（teatime）         | 代表 4:00 PM（相当于 16:00）。                               |
| 英文月名 日期 年份         | 比如 January 15 2018 表示 2018 年 1 月 15 号，年份可有可无。 |
| MMDDYY、MM/DD/YY、MM.DD.YY | 比如 011518 表示 2018 年 1 月 15 号。                        |
| now+时间                   | 以 minutes、hours、days 或 weeks 为单位，例如 now+5 days 表示命令在 5 天之后的此时此刻执行。 |


at 命令只要指定正确的时间，就可以输入需要在指定时间执行的命令。这个命令可以是系统命令，也可以是 Shell 脚本。

```shell
[root@Test-BK-Dev script]# cat hello.sh 
#!/bin/bash
echo "hello world!!"
[root@Test-BK-Dev script]# at now + 10 minutes
at> /script/hello.sh >> hello.log
at> <EOT>
job 2 at Thu Jul 15 15:41:00 2021
#按ctrl+D保存退出at
[root@Test-BK-Dev script]# at -c 2
#!/bin/sh
# atrun uid=0 gid=0
# mail root 0
umask 22
XDG_SESSION_ID=3205; export XDG_SESSION_ID
HOSTNAME=Test-BK-Dev; export HOSTNAME
SELINUX_ROLE_REQUESTED=; export SELINUX_ROLE_REQUESTED
SHELL=/bin/bash; export SHELL
…………………………内容省略
${SHELL:-/bin/sh} << 'marcinDELIMITER5a75d3e2'
/script/hello.sh >> hello.log
#可以看到第2个at任务内容
marcinDELIMITER5a75d3e2
```

【例 2】

```shell
[root@localhost ~J# at 02:00 2013-07-26
at> /bin/sync
at> /sbin/shutdown -h now
at> <EOT>
job 9 at 2013-07-26 02:00
\#在指定的时间关机。在一个at任务中是可以执行多个系统命令的
```

在使用系统定时任务时，不论执行的是系统命令还是 Shell 脚本，最好使用绝对路径来写命令，这样不容易报错。at 任务一旦使用 `Ctrl+D` 快捷键保存，实际上写入了 /var/spool/at/ 这个目录，这个目录内的文件可以直接被 atd 服务调用和执行。

 atq 命令和 atrm 命令。atq 命令用于查看当前等待运行的工作，atrm 命令后者用于删除指定的工作

```shell
[root@Test-BK-Dev script]# atq
2	Thu Jul 15 15:41:00 2021 a root
```

当前等待运行的工作

## 19、crontab命令：循环执行定时任务

crontab ：每天定时任务计划执行

crond 是 Linux 下用来周期地执行某种任务或等待处理某些事件的一个守护进程，crond 服务的启动和自启动方法如下：

```shell
[root@localhost ~]# service crond restart
停止 crond： [确定]
正在启动 crond： [确定]
#重新启动crond服务
[root@localhost ~]# chkconfig crond on
#设定crond服务为开机自启动
```

其实，在安装完成操作系统后，默认会安装 crond 服务工具，且 crond 服务默认就是自启动的。crond 进程每分钟会定期检查是否有要执行的任务，如果有，则会自动执行该任务。

接下来，在介绍 crontab 命令。该命令和 at 命令类似，也是通过 /etc/cron.allow 和 /etc/cron.deny 文件来限制某些用户是否可以使用 crontab 命令的。而且原则也非常相似：

- 当系统中有 /etc/cron.allow 文件时，只有写入此文件的用户可以使用 crontab 命令，没有写入的用户不能使用 crontab 命令。同样，如果有此文件，/etc/cron.deny 文件会被忽略，因为 /etc/cron.allow 文件的优先级更高。
- 当系统中只有 /etc/cron.deny 文件时，写入此文件的用户不能使用 crontab 命令，没有写入文件的用户可以使用 crontab 命令。
- 这个规则基本和 at 命令的规则一致，同样是 /etc/cron.allow 文件比 /etc/cron.deny 文件的优先级高，Linux 系统中默认只有 /etc/cron.deny 文件。

每个用户都可以实现自己的 crontab 定时任务，只需使用这个用户身份执行“crontab -e”命令即可。当然，这个用户不能写入 /etc/cron.deny 文件。

crontab 命令的基本格式如下：

```shell
[root@localhost ~]# crontab [选项] [file]
```

注意，这里的 file 指的是命令文件的名字，表示将 file 作为 crontab 的任务列表文件并载入 crontab，若在命令行中未指定文件名，则此命令将接受标准输入（键盘）上键入的命令，并将它们键入 crontab。

与此同时，下表罗列出了此命令常用的选项及功能。

| 选项    | 功能                                                         |
| ------- | ------------------------------------------------------------ |
| -u user | 用来设定某个用户的 crontab 服务，例如 "-u demo" 表示设备 demo 用户的 crontab 服务，此选项一般有 root 用户来运行。 |
| -e      | 编辑某个用户的 crontab 文件内容。如果不指定用户，则表示编辑当前用户的 crontab 文件。 |
| -l      | 显示某用户的 crontab 文件内容，如果不指定用户，则表示显示当前用户的 crontab 文件内容。 |
| -r      | 从 /var/spool/cron 删除某用户的 crontab 文件，如果不指定用户，则默认删除当前用户的 crontab 文件。 |
| -i      | 在删除用户的 crontab 文件时，给确认提示。                    |

其实 crontab 定时任务非常简单，只需执行“crontab -e”命令，然后输入想要定时执行的任务即可。不过，当我们执行“crontab -e”命令时，打开的是一个空文件，而且操作方法和 Vim 是一致的。

这个文件中是通过 5 个“*”来确定命令或任务的执行时间的，这 5 个“*”的具体含义如下表 所示。

| 项目      | 含义                           | 范围                    |
| --------- | ------------------------------ | ----------------------- |
| 第一个"*" | 一小时当中的第几分钟（minute） | 0~59                    |
| 第二个"*" | 一天当中的第几小时（hour）     | 0~23                    |
| 第三个"*" | 一个月当中的第几天（day）      | 1~31                    |
| 第四个"*" | 一年当中的第几个月（month）    | 1~12                    |
| 第五个"*" | 一周当中的星期几（week）       | 0~7（0和7都代表星期日） |

在时间表示中，还有一些特殊符号需要学习，如下 所示。

| 特殊符号    | 含义                                                         |
| ----------- | ------------------------------------------------------------ |
| *（星号）   | 代表任何时间。比如第一个"*"就代表一小时种每分钟都执行一次的意思。 |
| ,（逗号）   | 代表不连续的时间。比如"0 8，12，16***命令"就代表在每天的 8 点 0 分、12 点 0 分、16 点 0 分都执行一次命令。 |
| -（中杠）   | 代表连续的时间范围。比如"0 5 ** 1-6命令"，代表在周一到周六的凌晨 5 点 0 分执行命令。 |
| /（正斜线） | 代表每隔多久执行一次。比如"*/10****命令"，代表每隔 10 分钟就执行一次命令。 |

当“crontab -e”编辑完成之后，一旦保存退出，那么这个定时任务实际就会写入 /var/spool/cron/ 目录中，每个用户的定时任务用自己的用户名进行区分。而且 crontab 命令只要保存就会生效，只要 crond 服务是启动的。

| 时间               | 含义                                                         |
| ------------------ | ------------------------------------------------------------ |
| 45 22 *** 命令     | 在 22 点 45 分执行命令                                       |
| 0 17 ** 1 命令     | 在每周一的 17 点 0 分执行命令                                |
| 0 5 1，15** 命令   | 在每月 1 日和 15 日的凌晨 5 点 0 分执行命令                  |
| 40  4 ** 1-5 命令  | 在每周一到周五的凌晨 4 点 40 分执行命令                      |
| */10 4 *** 命令    | 在每天的凌晨 4 点，每隔 10 分钟执行一次命令                  |
| 0 0 1，15 * 1 命令 | 在每月 1 日和 15 日，每周一个 0 点 0 分都会执行命令，注意：星期几和几日最好不要同时出现，因为它们定义的都是天，非常容易让管理员混淆 |

在书写 crontab 定时任务时，需要注意以下几个事项：

- 6 个选项都不能为空，必须填写。如果不确定，则使用“*”代表任意时间。
- crontab 定时任务的最小有效时间是分钟，最大有效时间是月。像 2018 年某时执行、3 点 30 分 30 秒这样的时间都不能被识别。
- 在定义时间时，日期和星期最好不要在一条定时任务中出现，因为它们都以天为单位，非常容易让管理员混淆。
- 在定时任务中，不管是直接写命令，还是在脚本中写命令，最好都使用绝对路径。有时使用相对路径的命令会报错。

系统的crontab设置：

“crontab -e”是每个用户都可以执行的命令，也就是说，不同的用户身份可以执行自己的定时任务。但是有些定时任务需要系统执行，这时就需要编辑 /etc/crontab 这个配置文件了。

当然，并不是说写入 /etc/crontab 配置文件中的定时任务在执行时不需要用户身份，而是“crontab -e”命令在定义定时任务时，默认用户身份是当前登录用户。而在修改 /etc/crontab 配置文件时，定时任务的执行者身份是可以手工指定的。这样定时任务的执行会更加灵活，修改起来也更加方便。

修改此文件只能是root用户

## 20、anacron命令用法详解

anacron 是用来做什么的呢？设想这样一个场景，Linux 服务器会在周末关机两天，但是设定的定时任务大多在周日早上进行，但在这个时间点，服务器又处于关机状态，导致系统很多定时任务无法运行。

anacron 会以 1 天、1周（7天）、一个月作为检测周期，判断是否有定时任务在关机之后没有执行。如果有这样的任务，那么 anacron 会在特定的时间重新执行这些定时任务。

那么，anacron 是如何判断这些定时任务已经超过执行时间的呢？这就需要借助 anacron 读取的时间记录文件。anacron 会分析现在的时间与时间记录文件所记载的上次执行 anacron 的时间，将两者进行比较，如果两个时间的差值超过 anacron 的指定时间差值（一般是 1 天、7 天和一个月），就说明有定时任务没有执行，这时 anacron 会介入并执行这个漏掉的定时任务，从而保证在关机时没有执行的定时任务不会被漏掉。

anacron命令的基本格式如下：

```shell
[root@localhost ~]# anacron [选项] [工作名]
```

这里的工作名指的是依据 /etc/anacrontab 文件中定义的工作名。表 1 罗列出了此命令常用的几个选项及各自的功能。

| 选项 | 功能                                                         |
| ---- | ------------------------------------------------------------ |
| -f   | 强制执行相关工作，忽略时间戳。                               |
| -u   | 更新 /var/spool/anacron/cron.{daily，weekly，monthly} 文件中的时间戳为当前日期，但不执行任何工作。 |
| -s   | 依据 /etc/anacrontab 文件中设定的延迟时间顺序执行工作，在前一个工作未完成前，不会开始下一个工作。 |
| -n   | 立即执行 /etc/anacrontab 中所有的工作，忽略所有的延迟时间。  |
| -q   | 禁止将信息输出到标准错误，常和 -d 选项合用。                 |


在当前的 Linux 中，其实不需要执行任何 anacron 命令，只需要配置好 /etc/anacrontab 文件，系统就会依赖这个文件中的设定来通过 anacron 执行定时任务了，那么，关键就是 /etc/anacrontab 文件的内容了。这个文件的内容如下：

```shell
[root@Test-BK-Dev /]# cat /etc/anacrontab
# /etc/anacrontab: configuration file for anacron

# See anacron(8) and anacrontab(5) for details.

SHELL=/bin/sh
PATH=/sbin:/bin:/usr/sbin:/usr/bin
MAILTO=root
# the maximal random delay added to the base delay of the jobs
RANDOM_DELAY=45
#最大随机延迟
# the jobs will be started during the following hours only
START_HOURS_RANGE=3-22
#fanacron的执行时间范围是3:00~22:00
#period in days   delay in minutes   job-identifier   command
1	5	cron.daily		nice run-parts /etc/cron.daily
#每天开机 5 分钟后就检查 /etc/cron.daily 目录内的文件是否被执行，如果今天没有被执行，那就执行
7	25	cron.weekly		nice run-parts /etc/cron.weekly
#每隔 7 天开机后 25 分钟检查 /etc/cron.weekly 目录内的文件是否被执行，如果一周内没有被执行，就会执行
@monthly 45	cron.monthly		nice run-parts /etc/cron.monthly
#每隔一个月开机后 45 分钟检查 /etc/cron.monthly 目录内的文件是否被执行，如果一个月内没有被执行，那就执行 
```

在这个文件中，“RANDOM_DELAY”定义的是最大随机延迟，也就是说，cron.daily 工作如果超过 1 天没有执行，则并不会马上执行，而是先延迟强制延迟时间，再延迟随机延迟时间，之后再执行命令；“START_HOURS_RANGE”的是定义 anacron 执行时间范围，anacron 只会在这个时间范围内执行。

我们用 cron.daily 工作来说明一下 /etc/anacrontab 的执行过程:

1. 读取 /var/spool/anacron/cron.daily 文件中 anacron 上一次执行的时间。
2. 和当前时间比较，如果两个时间的差值超过 1 天，就执行 cron.daily 工作。
3. 只能在 03：00-22：00 执行这个工作。
4. 执行工作时强制延迟时间为 5 分钟，再随机延迟 0～45 分钟。
5. 使用 nice 命令指定默认优先级，使用 run-parts 脚本执行 /etc/cron.daily 目录中所有的可执行文件。

/etc/cron.{daily，weekly，monthly} 目录中的脚本在当前的 Linux 中是被 anacron 调用的，不再依靠 cron 服务。不过，anacron 不用设置多余的配置，我们只需要把需要定时执行的脚本放入 /etc/cron.{daily，weekly，monthly} 目录中，就会每天、每周或每月执行，而且也不再需要启动 anacron 服务了。如果需要进行修改，则只需修改 /etc/anacrontab 配置文件即可。

比如，我更加习惯让定时任务在凌晨 03：00-05：00 执行，就可以进行如下修改：

```shell
[root@localhost ~] # vi /etc/anacrontab
# /etc/anacrontab: configuration file for anacron
# See anacron(8) and anacrontab(5) for details.
SHELL=/bin/sh
PATH=/sbin:/bin:/usr/sbin:/usr/bin MAILTO-root
# the maximal random delay added to the base delay of the jobs 
RANDOM_DELAY=0
#把最大随机廷迟改为0分钟,不再随机廷迟
# the jobs will be started during the following hours only 
START_HOORS_RANGE=3-5
#执行时间范围为03:00—05:00
#period in days delay in minutes job-identifier command
1 0 cron.daily nice run-parts /etc/cron.daily
7 0 cron.weekly nice run-parts /etc/cron.weekly
@monthly 0 cron.monthly nice run-parts /etc/cron.monthly
#把强制延迟也改为0分钟,不再强制廷迟
```

这样，所有放入 /etc/cron.{daily，weekly，monthly} 目录中的脚本都会在指定时间执行，而且也不怕服务器万一关机的情况了。

## 21、vmstat命令详解：监控系统资源

如果你想动态的了解一下系统资源的使用状况，以及查看当前系统中到底是哪个环节最占用系统资源，就可以使用 vmstat 命令。

vmstat命令，是 Virtual Meomory Statistics（虚拟内存统计）的缩写，可用来监控 CPU 使用、进程状态、内存使用、虚拟内存使用、硬盘输入/输出状态等信息。此命令的基本格式有如下 2 种：

```shell
[root@localhost ~]# vmstat [-a] [刷新延时 刷新次数]
[root@localhost ~]# vmstat [选项] 
```

-a 的含义是用 inact/active（活跃与否） 来取代 buff/cache 的内存输出信息。除此之外，下表罗列出了 vmstat 命令的第二种基本格式中常用的选项及各自的含义。

| 选项              | 含义                                                         |
| ----------------- | ------------------------------------------------------------ |
| -fs               | -f：显示从启动到目前为止，系统复制（fork）的程序数，此信息是从 /proc/stat 中的 processes 字段中取得的。 -s：将从启动到目前为止，由一些事件导致的内存变化情况列表说明。 |
| -S 单位           | 令输出的数据显示单位，例如用 K/M 取代 bytes 的容量。         |
| -d                | 列出硬盘有关读写总量的统计表。                               |
| -p 分区设备文件名 | 查看硬盘分区的读写情况。                                     |

```shell
[root@gd-report-kettle etc]# vmstat 1 3   #1秒刷新一次，刷新3次
procs -----------memory----------  ---swap--   -----io---- -system--    ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo    in   cs      us sy id wa st
 1  0 1316996 11862108 20 3253584   0    0     1     4     0    0        1  1 98  0  0
 0  0 1316996 11861960 20 3253584   0    0     0     0     10079 18021   2  1 98  0  0
 0  0 1316996 11861960 20 3253584   0    0     0     0     9934 17714    1  1 98  0  0
```

| 字段   | 含义                                                         |
| ------ | ------------------------------------------------------------ |
| procs  | 进程信息字段：-r：等待运行的进程数，数量越大，系统越繁忙。-b：不可被唤醒的进程数量，数量越大，系统越繁忙。 |
| memory | 内存信息字段：-swpd：虚拟内存的使用情况，单位为 KB。-free：空闲的内存容量，单位为 KB。-buff：缓冲的内存容量，单位为 KB。-cache：缓存的内存容量，单位为 KB。 |
| swap   | 交换分区信息字段：-si：从磁盘中交换到内存中数据的数量，单位为 KB。-so：从内存中交换到磁盘中数据的数量，单位为 KB。这两个数越大，表明数据需要经常在磁盘和内存之间进行交换，系统性能越差。 |
| io     | 磁盘读/写信息字段：-bi：从块设备中读入的数据的总量，单位是块。-bo：写到块设备的数据的总量，单位是块。这两个数越大，代表系统的 I/O 越繁忙。 |
| system | 系统信息字段：-in：每秒被中断的进程次数。-cs：每秒进行的事件切换次数。这两个数越大，代表系统与接口设备的通信越繁忙。 |
| cpu    | CPU信息字段：-us：非内核进程消耗 CPU 运算时间的百分比。-sy：内核进程消耗 CPU 运算时间的百分比。-id：空闲 CPU 的百分比。-wa：等待 I/O 所消耗的 CPU 百分比。-st：被虚拟机所盗用的 CPU 百分比。 |

## 22、dmesg命令：显示开机信息

无论是系统启动过程中，还是系统运行过程中，只要是内核产生的信息，都会被存储在系统缓冲区中，如果开机时来不及查看相关信息，可以使用 dmesg 命令将信息调出，此命令常用于查看系统的硬件信息。

除此之外，开机信息也可以通过 /var/log/ 目录中的 dmesg 文件进行查看。

dmesg 命令的用法很简单，基本格式如下：

```shell
[root@localhost ~]# dmesg
```

## 23、free命令：查看内存使用状态

free 命令用来显示系统内存状态，包括系统物理内存、虚拟内存（swap 交换分区）、共享内存和系统缓存的使用情况，其输出和 top 命令的内存部分非常相似。
free 命令的基本格式如下：

```shell
[root@localhost ~]# free [选项]
```

表 1 罗列出了此命令常用的选项及各自的含义。

| 选项        | 含义                                                         |
| ----------- | ------------------------------------------------------------ |
| -b          | 以 Byte（字节）为单位，显示内存使用情况。                    |
| -k          | 以 KB 为单位，显示内存使用情况，此选项是 free 命令的默认选项。 |
| -m          | 以 MB 为单位，显示内存使用情况。                             |
| -g          | 以 GB 为单位，显示内存使用情况。                             |
| -t          | 在输出的最终结果中，输出内存和 swap 分区的总量。             |
| -o          | 不显示系统缓冲区这一列。                                     |
| -s 间隔秒数 | 根据指定的间隔时间，持续显示内存使用情况。                   |

```shell
[root@Test-BK-Dev etc]# free -m
              total        used        free      shared  buff/cache   available
Mem:          15849         524       14768          18         556       15028
Swap:         16383           0       16383
```

第一行显示的是各个列的列表头信息，各自的含义如下所示：

- total 是总内存数；
- used 是已经使用的内存数；
- free 是空闲的内存数；
- shared 是多个进程共享的内存总数；
- buffers 是缓冲内存数；
- cached 是缓存内存数。


Mem 一行指的是内存的使用情况；-/buffers/cache 的内存数，相当于第一行的 used-buffers-cached。+/buffers/cache 的内存数，相当于第一行的 free+buffers+cached；Swap 一行指的就是 swap 分区的使用情况。

## 24、w和who命令：查看登陆用户信息

Linux 中，使用 w 或 who 命令都可以查看服务器上目前已登录的用户信息，两者的区别在于，w 命令除了能知道目前已登陆的用户信息，还可以知道每个用户执行任务的情况。

首先，介绍一下 w 命令的使用，w 命令的基本格式如下：

[root@localhost ~]# w [选项] [用户名]

此命令常用选项及含义，如表 1 所示。如果 w 命令后跟 [用户名]，则表示只显示此用户的信息。

| 选项 | 含义                                            |
| ---- | ----------------------------------------------- |
| -h   | 不显示输出信息的标题                            |
| -l   | 用长格式输出                                    |
| -s   | 用短格式输出，不显示登陆时间，JCPU 和 PCPU 时间 |
| -V   | 显示版本信息                                    |

```shell
[root@Test-BK-Dev etc]# w
 09:35:08 up 2 days, 23:00,  3 users,  load average: 0.00, 0.01, 0.05
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU  WHAT
root     pts/0    10.6.115.133     Thu14    4.00s  0.10s  0.10s -bash
root     pts/1    10.6.115.133     09:31    3:40   0.40s  0.38s top
root     pts/2    10.6.115.133     09:31    3:16   0.02s  0.00s more /var/log/dmesg
```

上面的输出信息中，第一行其实和 top 命令的第一行非常类似，主要显示了当前的系统时间、系统从启动至今已运行的时间、登陆到系统中的用户数和系统平均负载。

平均负载（load average）指的是在 1 分钟、5 分钟、15 分钟内系统的负载状况。

从第二行开始，显示的是当前所有登陆系统的用户信息，第二行是用户信息的各列标题，从第三行开始每行代表一个用户。这些标题的含义如表 2 所示。

| 标题   | 含义                                                         |
| ------ | ------------------------------------------------------------ |
| USER   | 登录到系统的用户。                                           |
| TTY    | 登录终端。                                                   |
| FROM   | 表示用户从哪里登陆进来，一般显示远程登陆主机的 IP 地址或者主机名。 |
| LOGIN@ | 用户登陆的日期和时间。                                       |
| IDLE   | 表示某个程序上次从终端开始执行到现在所持续的时间。           |
| JCPU   | 和该终端连接的所有进程占用的 CPU 运算时间。这个时间里并不包括过去的后台作业时间，但是包括当前正在运行的后台作业所占用的时间。 |
| PCPU   | 当前进程所占用的 CPU 运算时间。                              |
| WHAT   | 当前用户正在执行的进程名称和选项，换句话说，就是表示用户当前执行的是什么命令。 |

相比较 w 命令，who 命令只能显示当前登陆的用户信心，但无法知晓每个用户正在执行的命令。 who 命令的基本格式如下：

```shell
[root@localhost ~]# who [选项] [file]
```

需要说明的是，who 命令默认是通过 /var/run/utmp 文件来获取登陆用户信息，但如果通过 file 指定另一个文件，则 who 命令将不再默认读取 /var/run/utmp 文件，而是读取该指定文件来获取信息。

有关 who 命令常用选项及含义，如表 3 所示。

| 选项     | 含义                                                         |
| -------- | ------------------------------------------------------------ |
| -a       | 列出所有信息，相当于所有选项。                               |
| -b       | 列出系统最近启动的时间日期。                                 |
| -l       | 列出所有可登陆的终端信息。                                   |
| -m       | 仅列出关于当前终端的信息，`who -m` 命令等同于 `who am i`。   |
| -q       | 列出在本地系统上的用户和用户数的清单。                       |
| -r       | 显示当前系统的运行级别。                                     |
| -s       | 仅显示名称、线路和时间字段信息，这是 who 命令的默认选项。    |
| -u       | 显示当前每个用户的用户名、登陆终端、登陆时间、线路活动和进程标识。 |
| -T 或 -w | 显示 tty 终端的状态，“+”表示对任何人可写，“-”表示仅对 root 用户或所有者可写，“？”表示遇到线路故障。 |

## 25、last和lastlog命令：查看过去登陆的用户信息

last 命令可以查看当前和过去登陆系统用户的相关信息；lastlog 命令可以查看到每个系统用户最近一次登陆系统的时间。

我们先来看看 last 命令，此命令的基本格式如下所示：

```
[root@localhost ~]# last [选项]
```

表 1 罗列出了该命令常用的选项及含义。

| 选项        | 含义                                               |
| ----------- | -------------------------------------------------- |
| -a          | 把从何处登陆系统的主机名或 IP 地址显示在最后一行。 |
| -R          | 不显示登陆系统的主机名或 IP 地址。                 |
| -x          | 显示系统关机、重新开机以及执行等级的改变等信息。   |
| -n 显示列数 | 设置列出信息的显示列数。                           |
| -d          | 将显示的 IP 地址转换成主机名称。                   |


在执行 last 命令时，它默认会读取 /var/log/wtmp 日志文件，这是一个二进制文件，不能直接用 vi 编辑，只能通过 last 命令调用。

lastlog 命令默认是去读取 /var/log/lastlog 日志文件的，这个文件同样是二进制文件，不能直接用 vi 编辑，需要使用 lastlog 命令调用。

# Linux系统服务管理

## 1、Linux系统服务及其分类

Linux 中的服务按照安装方法不同可以分为 RPM 包默认安装的服务和源码包安装的服务两大类。其中，RPM 包默认安装的服务又因为启动与自启动管理方法不同分为独立的服务和基于 xinetd 的服务。服务分类的关系图如图所示。

![](http://c.biancheng.net/uploads/allimg/181024/2-1Q02413195AP.jpg)

我们知道，Linux 中常见的软件包有两种：一种是 RPM 包；另一种是源码包。那么，通过 RPM 包安装的系统服务就是 RPM 包默认安装的服务（因为 Linux 光盘中全是 RPM 包，Linux 系统也是通过 RPM 包安装的，所以我们把 RPM 包又叫作系统默认包），通过源码包安装的系统服务就是源码包安装的服务。

源码包是开源的，自定义性强，通过编译安装更加适合系统，但是安装速度较慢，编译时容易报错。RPM 包是经过编译的软件包，安装更快速，不易报错，但不再是开源的。

以上这些特点都是软件包本身的特点，但是软件包一旦安装到 Linux 系统上，它们的区别是什么呢？

最主要的区别就是安装位置不同，源码包安装到我们手工指定的位置当中，而 RPM 包安装到系统默认位置当中（可以通过"rpm -ql 包名"命令查询）。也就是说，RPM 包安装到系统默认位置，可以被服务管理命令识别；但是源码包安装到手工指定位置，当然就不能被服务管理命令识别了（可以手工修改为被服务管理命令识别）。

所以，RPM 包默认安装的服务和源码包安装的服务的管理方法不同，我们把它们当成不同的服务分类。服务分类说明如下。

RPM 包默认安装的服务。这些服务是通过 RPM 包安装的，可以被服务管理命令识别。

这些服务又可以分为两种：

- 独立的服务：就是独立启动的意思，这种服务可以自行启动，而不用依赖其他的管理服务。因为不依赖其他的管理服务，所以，当客户端请求访问时，独立的服务响应请求更快速。目前，Linux 中的大多数服务都是独立的服务，如 apache 服务、FTP 服务、Samba 服务等。
- 基于 xinetd 的服务(需要yum install xinetd)：这种服务就不能独立启动了，而要依靠管理服务来调用。这个负责管理的服务就是 xinetd 服务。xinetd 服务是系统的超级守护进程，其作用就是管理不能独立启动的服务。当有客户端请求时，先请求 xinetd 服务，由 xinetd 服务去唤醒相对应的服务。当客户端请求结束后，被唤醒的服务会关闭并释放资源。这样做的好处是只需要持续启动 xinetd 服务，而其他基于 xinetd 的服务只有在需要时才被启动，不会占用过多的服务器资源。但是这种服务由于在有客户端请求时才会被唤醒，所以响应时间相对较长。


源码包安装的服务。这些服务是通过源码包安装的，所以安装位置都是手工指定的。由于不能被系统中的服务管理命令直接识别，所以这些服务的启动与自启动方法一般都是源码包设计好的。每个源码包的启动脚本都不一样，一般需要查看说明文档才能确定。

### 查询已经安装的服务和区分服务

我们已经知道 Linux 服务的分类了，那么应该如何区分这些服务呢？首先要区分 RPM 包默认安装的服务和源码包安装的服务。源码包安装的服务是不能被服务管理命令直接找到的，而且一般会安装到 /usr/local/ 目录中。

也就是说，在 /usr/local/ 目录中的服务都应该是通过源码包安装的服务。RPM 包默认安装的服务都会安装到系统默认位置，所以是可以被服务管理命令（如 service、chkconfig）识别的。

其次，在 RPM 包默认安装的服务中怎么区分独立的服务和基于 xinetd 的服务？这就要依靠 chkconfig 命令了。chkconfig 是管理 RPM 包默认安装的服务的自启动的命令，这里仅利用这条命令的查看功能。使用这条命令还能看到 RPM 包默认安装的所有服务。命令格式如下：

```
[root@localhost ~]# chkconfig --list [服务名]
```

- --list：列出 RPM 包默认安装的所有服务的自启动状态；

## 2、linux端口及查询方法

如果知道了一台服务器的 IP 地址，我们就可以找到这台服务器。但是这台服务器上有可能搭建了多个网络服务，比如 WWW 服务、FTP 服务、Mail 服务，那么我们到底需要服务器为我们提供哪个网络服务呢？这时就要靠端口（Port）来区分了，因为每个网络服务对应的端口都是固定的。

比如，WWW 服务对应的端口是 80，FTP 服务对应的端口是 20 和 21，Mail 服务对应的端口是 25 和 110。也就是说，IP 地址可以想象成"门牌号码"，而端口可以想象成"家庭成员"，找到了 IP 地址只能找到你们家，只有找到了端口，寄信时才能找到真正的收件人。

为了统一整个互联网的端口和网络服务的对应关系，以便让所有的主机都能使用相同的机制来请求或提供服务，同一个服务使用相同的端口，这就是协议。

计算机中的协议主要分为两大类：

- 面向连接的可靠的TCP协议（Transmission Control Protocol，传输控制协议）；
- 面向无连接的不可靠的UDP协议（User Datagram Protocol，用户数据报协议）；

这两种协议都支持 216，也就是 65535 个端口。这么多端口怎么记忆呢？系统给我们提供了服务与端口的对应文件 /etc/services。 

网络服务的端口能够修改吗？当然是可以的，不过一旦修改了端口，那么客户机在访问服务器时很难知道服务器对应的端口是什么，也就不能正确地获取服务了。所以，除非在实验环境下，否则不要修改网络服务对应的端口。

### 查询系统中已经启动的服务

既然每个网络服务对应的端口是固定的，那么是否可以通过查询服务器中开启的端口，来判断当前服务器开启了哪些服务？

当然是可以的。虽然判断服务器中开启的服务还有其他方法（如通过ps命令），但是通过端口的方法查看最为准确。命令格式如下：

```shell
[root@localhost ~]# netstat 选项
```

选项：

- -a：列出系统中所有网络连接，包括已经连接的网络服务、监听的网络服务和 Socket 套接字；
- -t：列出 TCP 数据；
- -u：列出 UDF 数据；
- -l：列出正在监听的网络服务（不包含已经连接的网络服务）；
- -n：用端口号来显示而不用服务名；
- -p：列出该服务的进程 ID (PID)；

```shell
[root@Test-BK-Dev /]# netstat -tlunp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 0.0.0.0:10050           0.0.0.0:*               LISTEN      1085/zabbix_agentd  
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      1068/sshd           
tcp6       0      0 :::10050                :::*                    LISTEN      1085/zabbix_agentd  
tcp6       0      0 :::3306                 :::*                    LISTEN      1406/mysqld         
tcp6       0      0 :::22                   :::*                    LISTEN      1068/sshd  
```

解释一下命令的执行结果：

- Proto：数据包的协议。分为 TCP 和 UDP 数据包；
- Recv-Q：表示收到的数据已经在本地接收缓冲，但是还没有被进程取走的数据包数量；
- Send-Q：对方没有收到的数据包数量；或者没有 Ack 回复的，还在本地缓冲区的数据包数量；
- Local Address：本地 IP : 端口。通过端口可以知道本机开启了哪些服务；
- Foreign Address：远程主机：端口。也就是远程是哪个 IP、使用哪个端口连接到本机。由于这条命令只能查看监听端口，所以没有 IP 连接到到本机；
- State:连接状态。主要有已经建立连接（ESTABLISED）和监听（LISTEN）两种状态，当前只能查看监听状态；
- PID/Program name：进程 ID 和进程命令；

```shell
[root@localhost ~]# netstat -an
#查看所有的网络连接，包括已连接的网络服务、监听的网络服务和Socket套接字
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address Foreign Address State
tcp 0 0 0.0.0.0:53575 0.0.0.0:* LISTEN
tcp 0 0 0.0.0.0:111 0.0.0.0:* LISTEN
tcp 0 0 0.0.0.0:22 0.0.0.0:* LISTEN
tcp 0 0 127.0.0.1:631 0.0.0.0:* LISTEN
tcp 0 0 127.0.0.1:25 0.0.0.0:* LISTEN
tcp 0 0 192.168.0.210:22 192.168.0.105:4868 ESTABLISHED
tcp 0 0 :::57454 :::* LISTEN
...省略部分输出...
udp 0 0 :::932 :::*
Active UNIX domain sockets (servers and established)
Proto RefCnt Flags Type State I-Node Path
#Socket套接字输出,后面有具体介绍
Unix 2 [ ACC ] STREAM LISTENING 11712 /var/run/dbus/system_bus_socket
unix 2 [ ACC ] STREAM LISTENING 8450 @/com/ubuntu/upstart unix 7. [ ] DGRAM 8651 @/org/kernel/udev/udevd
unix 2 [ ACC ] STREAM LISTENING 11942 @/var/run/hald/dbus-b4QVLkivf1
...省略部分输出...
```

执行"netstat -an"命令能査看更多的信息，在 Stated 中也看到了已经建立的连接（ESTABLISED）。这是 ssh 远程管理命令产生的连接，ssh 对应的端口是 22。

而且我们还看到了 Socket 套接字。在服务器上，除网络服务可以绑定端口，用端口来接收客户端的请求数据外，系统中的网络程序或我们自己开发的网络程序也可以绑定端口，用端口来接收客户端的请求数据。这些网络程序就是通过 Socket 套接字来绑定端口的。也就是说，网络服务或网络程序要想在网络中传递数据，必须利用 Socke 套接字绑定端口，并进行数据传递。

使用"netstat -an"命令查看到的这些 Socke 套接字虽然不是网络服务，但是同样会占用端口，并在网络中传递数据。

解释一下 Socket 套接字的输出：

- Proto：协议，一般是unix；
- RefCnt：连接到此Socket的进程数量；
- Flags：连接标识；
- Type：Socket访问类型；
- State：状态，LISTENING表示监听，CONNECTED表示已经建立连接；
- I-Node：程序文件的 i 节点号；
- Path：Socke程序的路径，或者相关数据的输出路径；

## 3、Linux独立服务管理（RPM包的启动与自启动）

### 独立服务的启动管理

独立的服务要想启动，主要有两种方法。

#### 1) 使用/etc/init.d/目录中的启动脚本来启动独立的服务

既然所有独立服务的启动脚本都存放在 /etc/init.d/ 目录中，那么，调用这些脚本就可以启动独立的服务了。这种启动方式是推荐启动方式，命令格式如下：

```shell
[root@localhost ~]#/etc/init.d独立服务名 start| stop|status|restart|...
```

参数：

- start：启动服务；
- stop：停止服务；
- status：查看服务状态；
- restart：重启动服务；

### 独立服务的自启动管理

自启动指的是在系统之后，服务是否随着系统的启动而自动启动。如果启动了某个服务，那么这个服务会在系统重启之后启动吗？

答案是不知道，因为启动命令只负责启动服务，而和服务的自启动完全没有关系。同样地，自启动命令只管服务是否会在系统重启之后启动，而和当前系统中的服务是否启动没有关系。

独立服务的自启动方法有三种，我们分别来学习。

#### 1) 使用 chkconfig 服务自启动管理命令

第一种方法是利用 chkconfig 服务自启动管理命令来管理独立服务的自启动，这条命令的用法如下：

[root@localhost ~]# chkconfig --list

使用 chkconfig 命令除了可以查看所有 RPM 包默认安装服务的自启动状态，也可以修改和设置 RPM 包默认安装服务的自启动状态，只是独立的服务和基于 xinetd 的服务的设定方法稍有不同。我们先来看看独立的服务如何设置。命令格式如下：

```shell
[root@localhost ~]# chkconfig [--level 运行级别][独立服务名][on|off]
```

\#选项：

- --level: 设定在哪个运行级别中开机自启动（on），或者关闭自启动（off）；

```shell
[root@localhost ~]# chkconfig --list | grep httpd
httpd 0:关闭 1:关闭 2:关闭 3:关闭 4:关闭 5:关闭 6:关闭
#查询httpd的自启动状态。所有的级别都是不自启动的
[root@localhost ~]# chkconfig --level 2345 httpd on
#设置apache服务在进入2、3、4、5级别时自启动
[root@localhost ~]# chkconfig --list | grep httpd
httpd 0:关闭 1:关闭 2:启用 3:启用 4:启用 5:启用 6:关闭
#查询apache服务的自启动状态。发现在2、3、4、5这4个运行级别中变为了"启用"
```

还记得 0~6 这 7 个 Linux 的运行级别吗？如果在 0~6 这 7 个运行级别中服务都显示"关闭"，则该服务不自启动。如果在某个运行级别中显示"启用"，则代表在进入这个运行级别时，该服务开机自启动。

#### 2) 修改 /etc/rc.d/rc.local 文件，设置服务自启动

第二种方法就是修改 /etc/rc.d/rc.local 文件，在文件中加入服务的启动命令。这个文件是在系统启动时，在输入用户名和密码之前最后读取的文件（注意：/etc/rc.d/rc.loca和/etc/rc.local 文件是软链接，修改哪个文件都可以）。这个文件中有什么命令，都会在系统启动时调用。

如果我们把服务的启动命令放入这个文件，这个服务就会在开机时自启动。命令如下：

```shell
[root@localhost ~]#vi /etc/rc.d/rc.local
#!/bin/sh
#
#This script will be executed *after* all the other init scripts.
#You can put your own initialization stuff in here if you don't want to do the full Sys V style init stuff.
touch /var/lock/subsys/local
/etc/rc.d/init.d/httpd start
#在文件中加入apache的启动命令
```

这样，只要重启之后，apache 服务就会开机自启动了。推荐大家使用这种方法管理服务的自启动，有两点好处：

- 第一，如果大家都采用这种方法管理服务的自启动，当我们碰到一台陌生的服务器时，只要查看这个文件就知道这台服务器到底自启动了哪些服务，便于集中管理。
- 第二，chkconfig 命令只能识别 RPM 包默认安装的服务，而不能识别源码包安装的服务。 源码包安装的服务的自启动也是通过 /etc/rc.d/rc.local 文件实现的，所以不会出现同一台服务器自启动了两种安装方法的同一个服务。


还要注意一下，修改 /etc/rc.d/rc.local 配置文件的自启动方法和 chkconfig 命令的自启动方法是两种不同的自启动方法。所以，就算通过修改 /etc/rc.d/rc.local 配置文件的方法让某个独立的服务自启动了，执行"chkconfig --list"命令并不到有什么变化。

#### 3) 使用 ntsysv 命令管理自启动

第三种方法是使用 ntsysv 命令调用窗口模式来管理服务的自启动，非常简单。命令格式如下：

[root@localhost ~]# ntsysv [--level 运行级别]

选项：

- --level 运行级别：可以指定设定自启动的运行级别；

例如：

```shell
[root@localhost ~]# ntsysv --level 235
#只设定2、3、5级别的服务自启动
[root@localhost ~]# ntsysv
#按默认的运行级别设置服务自启动
```

执行命令后，会和 setup 命令类似，出现命令界面，如图 1 所示。
![img](http://c.biancheng.net/uploads/allimg/181024/2-1Q02415591C13.jpg)
图 1 ntsysv命令界面
这个命令的操作是这样的：

- 上下键：在不同服务之间移动；

- 空格键：选定或取消服务的自启动。也就是在服务之前是否输入"*"；

- Tab键：在不同项目之间切换；

- F1键：显示服务的说明；

  需要注意的是，ntsysv 命令不仅可以管理独立服务的自启动，也可以管理基于 xinetd 服务的自启动。也就是说，只要是 RPM 包默认安装的服务都能被 ntsysv 命令管理。但是源码包安装的服务不行。

这样管理服务的自启动多么方便，为什么还要学习其他的服务自启动管理命令呢？ ntsysv 命令虽然简单，但它是红帽系列 Linux 的专有命令，其他的 Linux 发行版本不一定拥有这条命令，而且条命令也不能管理源码包安装的服务，所以我们推荐大家使用 /etc/rc.d/rc.local 文件来管理服务的自启动。

## 4、Linux基于xinetd服务的管理方法详解

## 基于 xinetd 服务的启动

基于 xinetd 的服务没有自己独立的启动脚本程序，是需要依赖 xinetd 的启动脚本来启动的。xinetd 本身是独立的服务，所以 xinetd 服务自己的启动方法和独立服务的启动方法是一致的。

但是，所有基于 xinetd 这个超级守护进程的其他服务就不是这样的了，必须修改该服务的配置文件，才能启动基于 xinetd 的服务。所有基于 xinetd 服务的配置文件都保存在 /etc/xinetd.d/ 目录中。

我们使用 Telnet 服务来举例。Telnet 服务是用来进行系统远程管理的，端口是 23。不过需要注意的是，Telnet 服务的远程管理数据在网络中是明文传输的，非常不安全，所以在生产服务器上是不建议启动 Telnet 服务的。在生产服务器上，远程管理使用的是 ssh 协议，ssh 协议是加密的，更加安全。

Telnet 服务也是分为"客户端/服务器端"的，其中服务器端是用来启动 Telnet 服务的，并不安全；客户端是用来连接服务器端或测试服务器的端口是否开启的，在实际工作中我们主要使用 Telnet 客户端来测试远程服务器开启了哪些端口。

既然基于 xinetd 服务的配置文件都在 /etc/xinetd.d/ 目录中，那么 Telnet 服务的配置文件就是 /etc/xinetd.d/telnet。我们打开这个文件看看，如下：

```shell
[root@localhost ~]#vi /etc/xinetd.d/telnet
#default: on
#description: The telnet server serves telnet sessions; it uses \
#unencrypted username/password pairs for authentication.
service telnet
#服务的名称为telnet
{
flags = REUSE
#标志为REUSE，设定TCP/IP socket可重用
socketjtype = stream
#使用 TCP协议数据包
wait = no
#允许多个连接同时连接
user = root
#启动服务的用户为root
server = /usr/sbin/in.telnetd
#服务的启动程序
log_on_failure += USERID
#登录失败后，记录用户的ID
disable = yes
#服务不启动
}
```

如果想要启动 Telnet 服务，则只需要把 /etc/xinetd.d/telnet 文件中的"disable=yes"改为"disable=no"即可，"disable"代表取消，"disable=yes"代表取消是 yes，当然不启动服务；"disable=no"代表取消是 no，当然就是启动服务了。具体命令如下：

```shell
[root@localhost ~]#vi /etc/xinetd.d/telnet
#修改配置文件
service telnet {
…省略部分输出…
disable = no
#把 yes 改为no
}
[root@localhost ~]# service xinetd restart
#重启xinetd服务
停止 xinetd:
[确定]
正在启动xinetd:
[确定]
[root@localhost ~]# netstat -tlun|grep 23
tcp 0 0 :::23 :::* LISTEN
#查看端口，23端口启动，表示Telne服务已经启动了
```

基于 xinetd 服务的启动都是这样的，只需修改 /etc/xinetd.d/ 目录中的配置文件，然后重启 xientd 服务即可。

### 基于xientd 服务的自启动

基于 xinetd 服务的自启动管理有两种方法，分别是通过 chkconfig 命令管理自启动和通过 ntsysv 命令管理自启动。但是不能通过修改 /etc/rc.d/rc.local 配置文件来管理自启动，因为基于 xinetd 的服务没有自己的启动脚本程序。我们分别来看看。

#### 1) 使用 chkconfig 命令管理自启动

chkconfig 自启动管理命令可以管理所有 RPM 包默认安装的服务，所以也可以用来管理基于 xinetd 服务的自启动。命令格式如下：

```shell
[root@localhost ~]# chkconfig 服务名 on|off
#基于xinetd的服务没有自己的运行级别，而依靠xinetd服务的运行级别，所以不用指定--level选项
```

```shell
[root@localhost ~]# chkconfig telnet on
#启动Telnet服务的自启动
[root@localhost ~]# chkconfig --list|grep telnet
telnet:启用
#查看服务的自启动，Telnet服务变为了"启用"
[root@localhost ~]# chkconfig telnet off
#关闭Telnet服务的自启动
[root@localhost ~]# chkconfig --list|grep telnet
telnet:关闭
#查看服务的自启动，Telnet服务变为了 "关闭"
```

#### 2) 使用 ntsysv 命令管理自启动

ntsysv 命令既然可以管理所有 RPM 包默认安装的服务，当然也能管理基于 xinetd 的服务。命令的使用方法和管理独立的服务是一样的，这里就不再重复介绍了。

其实，如果我们仔细来看，就会发现基于 xinetd 服务的启动和自启动区分得并不严格。启动命令也会把服务设置为自启动，自启动命令也会把服务设置为启动。我们做一个实验看看，命令如下：

```shell
[root@localhost ~]# vi /etc/xinetd.d/telnet service telnet
{
disable = yes
...省略部分输出...
}
[root@localhost ~]# service xinetd restart
停止xinetd: [确定]
正在启动xinetd: [确定】
[root@localhost ~]# chkconfig telnet off
#先关闭Telnet服务的启动和自启动，保证不会对后面的实验产生影响
[root@localhost ~]# vi /etc/xinetd.d/telnet service telnet
{
disable = no
...省略部分输出...
}
[root@localhost ~]# service xinetd restart
停止xinetd: [确定]
正在启动xinetd: [确定】
#然后启动Telnet服务
[root@localho.st ~] # chkconfig --list | grep telnet
telnet：启用
#看到了吗?我们一开始已经把Telnet服务的自启动关闭了。后面的实验虽然只启动了#Telnet服务，但是该服务自动变为了自启动状态
```

这个实验说明了基于 xinetd 服务的启动和自启动命令之间是通用的，在当前系统中启动了服务，服务的自启动也会开启；关闭了服务的自启动，当前系统中的服务也会关闭。

## 5、Linux源码包服务管理（启动与自启动）

### 源码包服务的启动管理

源码包服务中所有的文件都会安装到指定目录当中，并且没有任何垃圾文件产生（Linux 的特性），所以服务的管理脚本程序也会安装到指定目录中。源码包服务的启动管理方式就是在服务的安装目录中找到管理脚本，然后执行这个脚本。

问题来了，每个服务的启动脚本都是不一样的，我们怎么确定每个服务的启动脚本呢？还记得在安装源码包服务时，我们强调需要査看每个服务的说明文档吗（一般是 INSTALL 或 READEM）？在这个说明文档中会明确地告诉大家服务的启动脚本是哪个文件。

我们用 apache 服务来举例。一般 apache 服务的安装位置是 /usr/local/apache2/ 目录，那么 apache 服务的启动脚本就是 /usr/local/apache2/bin/apachectl 文件（查询 apache 说明文档得知）。启动命令如下：

```shell
[root@localhost ~]# /usr/local/apache2/bin/apachectl start|stop|restart|...
#源码包服务的启动管理
```

```shell
[root@localhost ~]# /usr/local/apache2/bin/apachectl start
#会启动源码包安装的apache服务
```

注意，不管是源码包安装的 apache，还是 RPM 包默认安装的 apache，虽然在一台服务器中都可以安装，但是只能启动一个，因为它们都会占用 80 端口。

源码包服务的启动方法就这一种，比 RPM 包默认安装的服务要简单一些。

### 源码包服务的自启动管理

源码包服务的自启动管理也不能依靠系统的服务管理命令，而只能把标准启动命令写入 /etc/rc.d/rc.local 文件中。系统在启动过程中读取 /etc/rc.d/rc.local 文件时，就会调用源码包服务的启动脚本，从而让该服务开机自启动。命令如下：

```shell
[root@localhost ~]# vi /etc/rc.d/rc.local
#修改自启动文件
#!/bin/sh
#This script will be executed *after* all the other init scripts.
#You can put your own initialization stuff in here if you don11
#want to do the full Sys V style init stuff.
touch /var/lock/subsys/local /usr/local/apache2/bin/apachectl start
#加入源码包服务的标准启动命令，保存退出，源码包安装的apache服务就被设为自启动了
```

### 让源码包服务被服务管理命令识别

不建议这样做，需要的话请度娘·~~~~~~~~~~

## 6、Linux常见服务类别及功能

| 服务名称        | 功能简介                                                     | 建议 |
| --------------- | ------------------------------------------------------------ | ---- |
| acpid           | 电源管理接口。如果是笔记本电脑用户，则建议开启，可以监听内核层的相关电源事件 | 开启 |
| anacron         | 系统的定时任务程序。是 cron 的一个子系统，如果定时任务错过了执行时间，则可以通过 anacron 继续唤醒执行 | 关闭 |
| alsasound       | alsa 声卡驱动。如果使用 alsa 声卡，则开启                    | 关闭 |
| apmd            | 电源管理模块。如果支持 acpid，就不需要 apmd，可以关闭        | 关闭 |
| atd             | 指定系统在特定时间执行某个任务，只能执行一次。如果需要则开启，但我们一般使用 crond 来执行循环定时任务 | 关闭 |
| auditd          | 审核子系统。如果开启了此服务，那么 SELinux 的审核信息会写入 /var/log/audit/ audit.log 文件；如果不开启，那么审核信息会记录在 syslog 中 | 开启 |
| autofs          | 让服务器可以自动挂载网络中其他服务器的共享数据,一般用来自动挂载 NFS 服务。如果没有 NFS 服务，则建议关闭 | 关闭 |
| avahi-daemon    | avahi 是 zeroconf 协议的实现，它可以在没有 DNS 服务的局域网里发现基于 zeroconf 协议的设备和服务。除非有兼容设备或使用 zeroconf 协议，否则关闭 | 关闭 |
| bluetooth       | 蓝牙设备支持。一般不会在服务器上启用蓝牙设备，关闭它         | 关闭 |
| capi            | 仅对使用 ISND 设备的用户有用                                 | 关闭 |
| chargen-dgram   | 使用 UDP 协议的 chargen server。其主要提供类似远程打字的功能 | 关闭 |
| chargen-stream  | 同上                                                         | 关闭 |
| cpuspeed        | 可以用来调整 CPU 的频率。当闲置时，可以自动降低 CPU 频率来节省电量 | 开启 |
| crond           | 系统的定时任务，一般的 Linux 服务器都需要定时任务来协助系统维护。建议开启 | 开启 |
| cvs             | 一个版本控制系统                                             | 关闭 |
| daytime-dgram   | 使用 TCP 协议的 daytime 守护进程，该协议为客户机实现从远程服务器获取日期和时间的功能 | 关闭 |
| daytime-slream  | 同上                                                         | 关闭 |
| dovecot         | 邮件服务中 POP3/IMAP 服务的守护进程，主要用来接收信件。如果启动了邮件服务则开启：否则关闭 | 关闭 |
| echo-dgram      | 服务器回显客户服务的进程                                     | 关闭 |
| echo-stream     | 同上                                                         | 关闭 |
| firstboot       | 系统安装完成后，有一个欢迎界面，需要对系统进行初始设定，这就是这个服务的作用。既然不是第一次启动了，则建议关闭 | 关闭 |
| gpm             | 在字符终端 (ttyl~tty6) 中可以使用鼠标复制和粘贴，这就是这个服务的功能 | 开启 |
| haldaemon       | 检测和支持 USB 设备。如果是服务器则可以关闭，个人机则建议开启 | 关闭 |
| hidd            | 蓝牙鼠标、键盘等蓝牙设备检测。必须启动 bluetooth 服务        | 关闭 |
| hplip           | HP 打印机支持，如果没有 HP 打印机则关闭                      | 关闭 |
| httpd           | apache 服务的守护进程。如果需要启动 apache，就开启           | 开启 |
| ip6tables       | IPv6 的防火墙。目前 IPv6 协议并没有使用，可以关闭            | 关闭 |
| iptables        | 防火墙功能。Linux 中的防火墙是内核支持功能。这是服务器的主要防护手段，必须开启 | 开启 |
| irda            | IrDA 提供红外线设备（笔记本电脑、PDA’s、手机、计算器等）间的通信支持。建议关闭 | 关闭 |
| irqbalance      | 支持多核处理器，让 CPU 可以自动分配系统中断（IRQ)，提高系统性能。目前服务器多是多核 CPU，请开启 | 开启 |
| isdn            | 使用 ISDN 设备连接网络。目前主流的联网方式是光纤接入和 ADSL，ISDN 己经非常少见，请关闭 | 关闭 |
| kudzu           | 该服务可以在开机时进行硬件检测，并会调用相关的设置软件。建议关闭，仅在需要时开启 | 关闭 |
| lvm2-monitor    | 该服务可以让系统支持LVM逻辑卷组，如果分区采用的是LVM方式，那么应该开启。建议开启 | 开启 |
| mcstrans        | SELinux 的支持服务。建议开启                                 | 开启 |
| mdmonitor       | 该服务用来监测 Software RAID 或 LVM 的信息。不是必需服务，建议关闭 | 关闭 |
| mdmpd           | 该服务用来监测 Multi-Path 设备。不是必需服务，建议关闭       | 关闭 |
| messagebus      | 这是 Linux 的 IPC (Interprocess Communication，进程间通信）服务，用来在各个软件中交换信息。建议关闭 | 关闭 |
| microcode _ctl  | Intel 系列的 CPU 可以通过这个服务支持额外的微指令集。建议关闭 | 关闭 |
| mysqld          | [MySQL](http://c.biancheng.net/mysql/) 数据库服务器。如果需要就开启；否则关闭 | 开启 |
| named           | DNS 服务的守护进程，用来进行域名解析。如果是 DNS 服务器则开启；否则关闭 | 关闭 |
| netfs           | 该服务用于在系统启动时自动挂载网络中的共享文件空间，比如 NFS、Samba 等。 需要就开启，否则关闭 | 关闭 |
| network         | 提供网络设罝功能。通过这个服务来管理网络，建议开启           | 开启 |
| nfs             | NFS (Network File System) 服务，Linux 与 Linux 之间的文件共享服务。需要就开启，否则关闭 | 关闭 |
| nfslock         | 在 Linux 中如果使用了 NFS 服务，那么，为了避免同一个文件被不同的用户同时编辑，所以有这个锁服务。有 NFS 时开启，否则关闭 | 关闭 |
| ntpd            | 该服务可以通过互联网自动更新系统时间.使系统时间永远准确。需要则开启，但不是必需服务 | 关闭 |
| pcscd           | 智能卡检测服务，可以关闭                                     | 关闭 |
| portmap         | 用在远程过程调用 (RPC) 的服务，如果没有任何 RPC 服务，则可以关闭。主要是 NFS 和 NIS 服务需要 | 关闭 |
| psacct          | 该守护进程支持几个监控进程活动的工具                         | 关闭 |
| rdisc           | 客户端 ICMP 路由协议                                         | 关闭 |
| readahead_early | 在系统开启的时候，先将某些进程加载入内存整理，可以加快启动速度 | 关闭 |
| readahead_later | 同上                                                         | 关闭 |
| restorecond     | 用于给 SELinux 监测和重新加载正确的文件上下文。如果开启 SELinux，则需要开启 | 关闭 |
| rpcgssd         | 与 NFS 有关的客户端功能。如果没有 NFS 就关闭                 | 关闭 |
| rpcidmapd       | 同上                                                         | 关闭 |
| rsync           | 远程数据备份守护进程                                         | 关闭 |
| sendmail        | sendmail 邮件服务的守护进程。如果有邮件服务就开启；否则关闭  | 关闭 |
| setroubleshoot  | 该服务用于将 SELinux 相关信息记录在日志 /var/log/messages 中。建议开启 | 开启 |
| smartd          | 该服务用于自动检测硬盘状态。建议开启                         | 开启 |
| smb             | 网络服务 samba 的守护进程。可以让 Linux 和 Windows 之间共享数据。如果需要则开启 | 关闭 |
| squid           | 代理服务的守护进程。如果需要则开启：否则关闭                 | 关闭 |
| sshd            | ssh 加密远程登录管理的服务。服务器的远程管理必须使用此服务，不要关闭 | 开启 |
| syslog          | 日志的守护进程                                               | 开启 |
| vsftpd          | vsftp 服务的守护进程。如果需要 FTP 服务则开启；否则关闭      | 关闭 |
| xfs             | 这是 X Window 的字体守护进程，为图形界面提供字体服务。如果不启动图形界面，就不用开启 | 关闭 |
| xinetd          | 超级守护进程。如果有依赖 xinetd 的服务，就必须开启           | 开启 |
| ypbind          | 为 NIS (网络信息系统）客户机激活 ypbind 服务进程             | 关闭 |
| yum-updatesd    | yum 的在线升级服务                                           | 关闭 |

## 7、影响Linux系统性能的因素有哪些？

### CPU

CPU 是操作系统稳定运行的根本，CPU 的速度与性能很大一部分决定了系统整体的性能，因此 CPU 数量越多、主频越高，服务器性能也就相对越好。

但亊实也并非完全如此，目前大部分 CPU 在同一时间内只能运行一个线程，超线程的处理器可以在同一时间运行多个线程，因而可以利用处理器的超线程特性提髙系统性能。

而在 Linux 系统下，只有运行 SMP 内核才能支持超线程，但是安装的 CPU 数量越多，从超线程获得的性能上的提高就越少。另外，Linux 内核会把多核的处理器当作多个单独的 CPU 来识别，例如两颗 4 核的 CPU 在 Linux 系统下会认为是 8 颗 CPU。

> <!--注意，从性能角度来讲，两颗 4 核的 CPU 和 8 颗单核的 CPU 并不完全等价，根据权威部门得出的测试结论，前者的整体性能要低于后者 25%〜30%。-->

在 Linux 系统中，邮件服务器、动态 Web 服务器等应用对 CPU 性能的要求相对较高，因此对于这类应用，要把 CPU 的配置和性能放在主要位置。

### 内存

内存的大小也是影响 Linux 性能的一个重要的因素。内存太小，系统进程将被阻塞，应用也将变得缓慢，甚至失去响应；内存太大，会导致资源浪费。

Linux 系统采用了物理内存和虚拟内存的概念，虚拟内存虽然可以缓解物理内存的不足，但是占用过多的虚拟内存，应用程序的性能将明显下降。要保证应用程序的高性能运行，物理内存一定要足够大，但不应过大，否则会造成内存资源的浪费。

例如，在一个 32 位处理器的 Linux 操作系统上，超过 8GB 的物理内存都将被浪费。因此，要使用更大的内存，建议安装 64 位的操作系统，同时开启 Linux 的大内存内核支持。

不仅如此，由于处理器寻址范围的限制，在 32 位 Linux 操作系统上，应用程序单个进程最大只能使用 2GB 的内存。这样即使系统有更大的内存，应用程序也无法“享”用，解决的办法就是使用 64 位处理器，安装 64 位操作系统，在 64 位操作系统下，可以满足所有应用程序对内存的使用需求，几乎没有限制。

对内存性能要求比较的应用有打印服务器、数据库服务器和静态 Web 服务器等，因此对于这类应用，要把内存大小放在主要位置。

### 磁盘读写（I/O）能力

磁盘的 I/O 能力会直接影响应用程序的性能。比如说，在一个需要频繁读写的应用中，如果磁盘 I/O 性能得不到满足，就会导致应用的停滞。

不过，好在现今的磁盘都采用了很多方法来提高 I/O 性能，比如常见的磁盘 RAID 技术。

RAID 的英文全称为 Redundant Array of Independent Disks，翻译成中文为独立磁盘冗余阵列，简称磁盘阵列。RAID 通过把多块独立的磁盘（物理硬盘）按不同方式组合起来，形成一个磁盘组（逻辑硬盘），从而提供比单个硬盘更高的 I/O 性能和数据冗余。

通过 RAID 技术组成的磁盘组，就相当于一个大硬盘，用户可以对它进行分区格式化、建立文件系统等操作，跟单个物理硬盘一模一样，惟一不同的是 RAID 磁盘组的 I/O 性能比单个硬盘要高很多，同时对数据的安全性也有很大提升。

> <!--有关 RAID 更多的介绍，可阅读《[Linux RAID（磁盘列阵）完全攻略](http://c.biancheng.net/view/vip_5093.html)》一节做深入了解。-->

### 网络带宽

Linux 下的各种应用，一般都是基于网络的，因此网络带宽也是影响性能的一个重要因素，低速的、不稳定的网络将导致网络应用程序的访问阻塞；而稳定、高速的带宽，可以保证应用程序在网络上畅通无阻地运行。

幸运的是，现在的网络一般都是千兆带宽，或者光纤网络，带宽问题对应用程序性能造成的影响也在逐步降低。

通过对以上 4 个方面的讲述，不难看出，各个方面之间都是相互依赖的，不能孤立地从某个方面来排查问题。换句话说，当一个方面出现性能问题时，往往会引发其他方面出现问题。

例如，大量的磁盘读写势必消耗 CPU 和 I/O 资源，而内存的不足会导致频繁地进行内存页写入磁盘、磁盘写到内存的操作，造成磁盘 I/O 瓶颈，同时大量的网络流量也会造成 CPU 过载。总之，在处理性能问题时，应纵观全局，从各个方面进行综合考虑。

## 8、sar命令详解：分析系统性能

sar 命令的基本格式如下：

```shell
[root@localhost ~]# sar [options] [-o filename] interval [count]
```

此命令格式中，各个参数的含义如下：

- -o filename：其中，filename 为文件名，此选项表示将命令结果以二进制格式存放在文件中；
- interval：表示采样间隔时间，该参数必须手动设置；
- count：表示采样次数，是可选参数，其默认值为 1；
- options：为命令行选项，由于 sar 命令提供的选项很多，这里不再一一介绍，仅列举出常用的一些选项及对应的功能，如表 1 所示。

| sar命令选项 | 功能                                                         |
| ----------- | ------------------------------------------------------------ |
| -A          | 显示系统所有资源设备（CPU、内存、磁盘）的运行状况。          |
| -u          | 显示系统所有 CPU 在采样时间内的负载状态。                    |
| -P          | 显示当前系统中指定 CPU 的使用情况。                          |
| -d          | 显示系统所有硬盘设备在采样时间内的使用状态。                 |
| -r          | 显示系统内存在采样时间内的使用情况。                         |
| -b          | 显示缓冲区在采样时间内的使用情况。                           |
| -v          | 显示 inode 节点、文件和其他内核表的统计信息。                |
| -n          | 显示网络运行状态，此选项后可跟 DEV（显示网络接口信息）、EDEV（显示网络错误的统计数据）、SOCK（显示套接字信息）和 FULL（等同于使用 DEV、EDEV和SOCK）等，有关更多的选项，可通过执行 man sar 命令查看。 |
| -q          | 显示运行列表中的进程数、进程大小、系统平均负载等。           |
| -R          | 显示进程在采样时的活动情况。                                 |
| -y          | 显示终端设备在采样时间的活动情况。                           |
| -w          | 显示系统交换活动在采样时间内的状态。                         |

> 有关 sar 命令更多可用的选项及功能，可通过执行 man sar 命令查看。

## 9、Linux如何查看CPU运行状态（详解版）

### a）Linux CPU性能分析：vmstat命令

vmstat 命令可以显示关于系统各种资源之间相关性能的简要信息，在 《[Linux vmstat命令](http://c.biancheng.net/view/1081.html)》一节中，我们已经对此命令的基本格式和用法做了详细的介绍，因此不再赘述，这里主要用它来看 CPU 的一个负载情况。

下面是 vmstat 命令在当前测试系统中的输出结果：

```
[root@localhost ~]# vmstat 2 3
procs -----------memory---------- ---swap-- -----io---- --system-- -----cpu-----
r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
1  0      8 247512  39660 394168    0    0    31     8   86  269  0  1 98  1  0
0  0      8 247480  39660 394172    0    0     0     0   96  147  4  0 96  0  0
0  0      8 247484  39660 394172    0    0     0    66   95  141  2  2 96  0  0
```

通过分析 vmstat 命令的执行结果，可以获得一些与当前 Linux 运行性能相关的信息。比如说：

- r 列表示运行和等待 CPU 时间片的进程数，如果这个值长期大于系统 CPU 的个数，就说明 CPU 不足，需要增加 CPU。
- swpd 列表示切换到内存交换区的内存数量（以 kB 为单位）。如果 swpd 的值不为 0，或者比较大，而且 si、so 的值长期为 0，那么这种情况下一般不用担心，不用影响系统性能。
- cache 列表示缓存的内存数量，一般作为文件系统缓存，频繁访问的文件都会被缓存。如果缓存值较大，就说明缓存的文件数较多，如果此时 I/O 中 bi 比较小，就表明文件系统效率比较好。
- 一般情况下，si（数据由硬盘调入内存）、so（数据由内存调入硬盘） 的值都为 0，如果 si、so 的值长期不为 0，则表示系统内存不足，需要增加系统内存。
- 如果 bi+bo 的参考值为 1000 甚至超过 1000，而且 wa 值较大，则表示系统磁盘 I/O 有问题，应该考虑提高磁盘的读写性能。
- 输出结果中，CPU 项显示了 CPU 的使用状态，其中当 us 列的值较高时，说明用户进程消耗的 CPU 时间多，如果其长期大于 50%，就需要考虑优化程序或算法；sy 列的值较高时，说明内核消耗的 CPU 资源较多。通常情况下，us+sy 的参考值为 80%，如果其值大于 80%，则表明可能存在 CPU 资源不足的情况。


总的来说，vmstat 命令的输出结果中，我们应该重点注意 procs 项中 r 列的值，以及 CPU 项中 us 列、sy 列和 id 列的值。

### b）Linux CPU性能分析：sar命令

除了 vmstat 命令，sar 命令也可以用来检查 CPU 性能，它可以对系统的每个方面进行单独的统计。

> 注意，虽然使用 sar 命令会增加系统开销，不过这些开销是可以评估的，不会对系统性能的统计结果产生很大影响。

和 vmstat 命令一样，sar 命令的基本格式和用法已经在 《[Linux sar命令](http://c.biancheng.net/view/6212.html)》一节中做了详细的介绍，接下来直接学习如何使用 sar 命令查看 CPU 性能。

下面是 sar 命令对当前测试系统的 CPU 统计输出结果：

```
[root@localhost ~]# sar -u 3 5
Linux 2.6.32-431.el6.x86_64 (localhost)  10/28/2019  _x86_64_ (8 CPU)

04:02:46 AM     CPU     %user     %nice   %system   %iowait    %steal     %idle
04:02:49 AM     all      1.69      0.00      2.03      0.00      0.00     96.27
04:02:52 AM     all      1.68      0.00      0.67      0.34      0.00     97.31
04:02:55 AM     all      2.36      0.00      1.69      0.00      0.00     95.95
04:02:58 AM     all      0.00      0.00      1.68      0.00      0.00     98.32
04:03:01 AM     all      0.33      0.00      0.67      0.00      0.00     99.00
Average:        all      1.21      0.00      1.35      0.07      0.00     97.37
```

此输出结果统计的是系统中包含的 8 颗 CPU 的整体运行状况，每项的输出都非常直观，其中最后一行（Average）是汇总行，是对上面统计信息的一个平均值。

> 需要指出的是，sar 输出结果中第一行包含了 sar 命令本身的统计消耗，因此 %user 列的值会偏高一点，但这并不会对统计结果产生很大影响。

另外，在一个多 CPU 的系统中，如果程序使用了单线程，就会出现“CPU 整体利用率不高，但系统应用响应慢”的现象，造成此现象的原因在于，单线程只使用一个 CPU，该 CPU 占用率为 100%，无法处理其他请求，但除此之外的其他 CPU 却处于闲置状态，进而整体 CPU 使用率并不高。

针对这个问题，可以使用 sar 命令单独查看系统中每个 CPU 的运行状态，例如：

```
[root@localhost ~]# sar -P 0 3 5
Linux 2.6.32-431.el6.x86_64 (localhost)  10/28/2019  _x86_64_ (8 CPU)

04:44:57 AM     CPU     %user     %nice   %system   %iowait    %steal     %idle
04:45:00 AM       0      8.93      0.00      1.37      0.00      0.00     89.69
04:45:03 AM       0      6.83      0.00      1.02      0.00      0.00     92.15
04:45:06 AM       0      0.67      0.00      0.33      0.33      0.00     98.66
04:45:09 AM       0      0.67      0.00      0.33      0.00      0.00     99.00
04:45:12 AM       0      2.38      0.00      0.34      0.00      0.00     97.28
Average:          0      3.86      0.00      0.68      0.07      0.00     95.39
```

注意，sar 命令对系统中 CPU 的计数是从数字 0 开始的，因此上面执行的命令表示对系统中第一颗 CPU 的运行状态进行统计。如果想单独统计系统中第 5 颗 CPU 的运行状态，可以执行`sar -P 4 3 5`命令。

### c）Linux CPU性能分析：iostat命令

iostat 命令主要用于统计磁盘 I/O 状态，但也能用来查看 CPU 的使用情况，只不过使用此命令仅能显示系统所有 CPU 的平均状态，无法向 sar 命令那样做具体分析。

使用 iostat 命令查看 CPU 运行状态，需要使用该命令提供的 -c 选项，该选项的作用是仅显示系统 CPU 的运行情况。例如：

```
[root@localhost ~]# iostat -c
Linux 2.6.32-431.el6.x86_64 (localhost)  10/28/2019  _x86_64_ (8 CPU)

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           0.07    0.00    0.12    0.09    0.00   99.71
```

可以看到，此输出结果中包含的项和 sar 命令的输出项完全相同。

> 有关 iostat 命令的基本用法，由于不是本节重点，因为不再详细介绍。

### d）Linux CPU性能分析：uptime命令

uptime 命令是监控系统性能最常用的一个命令，主要用来统计系统当前的运行状况。例如：

```
[root@localhost ~]# uptime
05:38:26 up  1:47,  2 users,  load average: 0.12, 0.08, 0.08
```

该命令的输出结果中，各个数据所代表的含义依次是：系统当前时间、系统运行时长、当前登陆系统的用户数量、系统分别在 1 分钟、5 分钟和 15 分钟内的平均负载。

这里需要注意的是，load average 这 3 个输出值一般不能大于系统 CPU 的个数。例如，本测试系统有 8 个 CPU，如果 load average 中这 3 个值长期大于 8，就说明 CPU 很繁忙，负载很高，系统性能可能会受到影响；如果偶尔大于 8 则不用担心，系统性能一般不会受到影响；如果这 3 个值小于 CPU 的个数（如本例所示），则表示 CPU 是非常空闲的。

总的来说，本节介绍了 4 个可查看 CPU 性能的命令，但这些命令也仅能查看 CPU 是否繁忙、负载是否过大，无法知道造成这种现象的根本原因。因此，在明确判断出系统 CPU 出现问题之后，还要结合 top、ps 等命令，进一步检查出是哪些进程造成的。

另外要知道的是，引发 CPU 资源紧张的原因有多个，可能是应用程序不合理造成的，也可能是硬件资源匮乏引起的。因此，要学会具体问题具体分析，或者优化应用程序，或者增加系统 CPU 资源。

## 10、Linux如何查看内存的使用情况？

## 11、Linux如何查看硬盘的读写性能？

### a）Linux查看硬盘读写性能：sar -d命令

《[Linux sar命令](http://c.biancheng.net/view/6212.html)》一节，已经对 sar 命令的基本用法做了详细的介绍，这里不再赘述，接下来主要讲解如何通过 sar -d 命令分析出硬盘读写的性能。

下面是执行 sar -d 命令的输出结果样例：

```
[root@localhost ~]# sar -d 3 5
Linux 2.6.32-431.el6.x86_64 (localhost)     10/25/2019     _x86_64_    (1 CPU)

06:36:52 AM       DEV       tps  rd_sec/s  wr_sec/s  avgrq-sz  avgqu-sz     await     svctm     %util
06:36:55 AM    dev8-0      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00

06:36:55 AM       DEV       tps  rd_sec/s  wr_sec/s  avgrq-sz  avgqu-sz     await     svctm     %util
06:36:58 AM    dev8-0      1.00      0.00     12.00     12.00      0.00      0.00      0.00      0.00

06:36:58 AM       DEV       tps  rd_sec/s  wr_sec/s  avgrq-sz  avgqu-sz     await     svctm     %util
06:37:01 AM    dev8-0      1.99      0.00     47.76     24.00      0.00      0.50      0.25      0.05

Average:          DEV       tps  rd_sec/s  wr_sec/s  avgrq-sz  avgqu-sz     await     svctm     %util
Average:       dev8-0      1.00      0.00     19.97     20.00      0.00      0.33      0.17      0.02
```

结合以上输出结果，可以遵循如下标准来判断当前硬盘的读写（I/O）性能：

- 通常情况下 svctm 的大小和硬盘性能有关，其值小于 await。但需要注意的是，CPU、内存的负荷也会对 svctm 的值造成影响，过多的请求也会间接导致 svctm 值的增加。
- await 值通常会受到 svctm、I/O 队列长度以及 I/O 请求模式的影响，如果 svctm 的值和 await 很接近，则表示几乎没有 I/O 等待，当前硬盘的性能很好；如果 await 的值远高于 svctm，则表示 I/O 队列等待太长，系统上运行的应用程序将变慢，此时可以通过更换更快的硬盘来解决问题。
- %util 项也是衡量硬盘 I/O 性能的重要指标，即如果其值接近 100%，就表示硬盘产生的 I/O 请求太多，I/O 系统正在满负荷工作，长期这样会影响系统的性能。必要时，可以优化程序或者更换更大、更快的硬盘来解决这个问题。

### b）Linux查看硬盘读写性能：iostat -d命令

通过执行 iostat -d 命令，也可以查看系统中硬盘的使用情况，如下是执行此命令的一个样例输出：

```
[root@localhost ~]# iostat -d 2 3
Linux 2.6.32-431.el6.x86_64 (localhost)  10/30/2019  _x86_64_ (8 CPU)

Device:            tps   Blk_read/s   Blk_wrtn/s   Blk_read   Blk_wrtn
sda               5.29       337.11         9.51     485202      13690

Device:            tps   Blk_read/s   Blk_wrtn/s   Blk_read   Blk_wrtn
sda               1.00         8.00        16.00         16         32

Device:            tps   Blk_read/s   Blk_wrtn/s   Blk_read   Blk_wrtn
sda               0.00         0.00         0.00          0          0
```

此输出结果中，我们重点看后面 4 列，它们各自表示的含义分别是：

- Blk_read/s：表示每秒读取的数据块数；
- Blk_wrtn/s：表示每秒写入的数据块数；
- Blk_read：表示读取的所有块数；
- Blk_wrtn：表示写入的所有块数。

> 注意，此输出结果中，第一次输出的数据是系统从启动以来直到统计时的所有传输信息，从第二次输出的数据开始，才代表在指定检测时间段内系统的传输值。

根据 iostat 命令的输出结果，我们也可以从中判断出当前硬盘的 I/O 性能。比如说，如果 Blk_read/s 的值很大，就表示当前硬盘的读操作很频繁；同样，如果 Blk_wrtn/s 的值很大，就表示当前硬盘的写操作很频繁。对于 Blk_read 和 Blk_wrtn 的大小，没有一个固定的界限，不同的系统应用对应值的范围也不同。但如果长期出现超大的数据读写情况，通常是不正常的，一定程度上会影响系统性能。

不仅如此，iostat 命令还提供了统计指定硬盘 I/O 状况的方法，即使用 -x 选项。例如：

```
[root@localhost ~]# iostat -x /dev/sda 2 3
Linux 2.6.32-431.el6.x86_64 (localhost)  10/30/2019  _x86_64_ (8 CPU)

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           0.09    0.00    0.24    0.26    0.00   99.42

Device:         rrqm/s   wrqm/s     r/s     w/s   rsec/s   wsec/s avgrq-sz avgqu-sz   await  svctm  %util
sda               2.00     0.56    2.92    0.44   206.03     7.98    63.56     0.03    9.99   5.31   1.79

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           0.31    0.00    0.06    0.00    0.00   99.62

Device:         rrqm/s   wrqm/s     r/s     w/s   rsec/s   wsec/s avgrq-sz avgqu-sz   await  svctm  %util
sda               0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00   0.00   0.00

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           0.50    0.00    0.06    0.00    0.00   99.44

Device:         rrqm/s   wrqm/s     r/s     w/s   rsec/s   wsec/s avgrq-sz avgqu-sz   await  svctm  %util
sda               0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00   0.00   0.00
```

此输出结果基本和 sar -d 命令的输出结果相同，需要额外说明的几个选项的含义如下：

- rrqm/s：表示每秒被合并的读操作数目（文件系统会对读取同一 block 块的请求进行合并）
- wrqm/s：表示每秒被合并的写操作数目；
- r/s：表示每秒完成读 I/O 设备的次数；
- w/s：表示每秒完成写 I/O 设备的次数；
- rsec/s：表示每秒读取的扇区数；
- wsec/s：表示每秒写入的扇区数；

### c）Linux查看硬盘读写性能：vmstat -d命令

使用 vmstat 命令也可以查看有关硬盘的统计数据，例如：

```
[root@bogon ~]# vmstat -d 3 2
disk- ------------reads------------ ------------writes----------- -----IO------
       total merged sectors      ms  total merged sectors      ms    cur    sec
sda     6897   4720  485618   71458   1256   1475   21842    9838      0     43
disk- ------------reads------------ ------------writes----------- -----IO------
       total merged sectors      ms  total merged sectors      ms    cur    sec
sda     6897   4720  485618   71458   1256   1475   21842    9838      0     43
```

该命令的输出结果显示了硬盘的读（reads）、写（writes）以及 I/O 的使用状态。

以上主要讲解了如何通过命令查看当前系统中硬盘 I/O 的性能，其实影响硬盘 I/O 的因素是多方面的，例如应用程序本身、硬件设计、系统自身配置等等。

要想解决硬盘 I/O 的瓶颈，关键是要提高 I/O 子系统的执行效率。比如说，首先从应用程序上对硬盘读写性能进行优化，能够放到内存中执行的操作尽量别保存到硬盘里（内存读写效率要远高于硬盘读写效率）；其次，还可以对硬盘存储方法进行合理规划，选择合适的 RAID 存储方式；最后，选择适合自身应用的文件系统，必要时可以使用裸设备提高硬盘的读写性能。

在裸设备上，数据可以直接读写，不必经过操作系统级别的缓存，还可以避免文件系统级别的维护开销（文件系统需要维护超级块、I-node 块等）以及操作系统的 cache 预读功能（减少了 I/O 请求）。
