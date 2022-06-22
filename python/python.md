# 1、python3的安装

## 1.1 修改yum源

```shell
wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
```

## 1.2 安装依赖

```shell
yum -y install zlib-devel bzip2-devel openssl-devel ncurses-devel sqlite-devel readline-devel tk-devel gcc make
```

```shell
yum install libffi-devel wget  xz  atuomake   epel-release git -y
```

## 1.3 编译安装

```shell
./configure --prefix=/user/local/python3
make
make install
```

## 1.4 创建软链接

```shell
ln -s /usr/local/python3/bin/python3.8 /usr/local/bin/python3
```

## 1.5 添加环境变量

```shell
vim .bash_profile
在文件中添加：
export PYTHON_HOME=/usr/local/python3/bin/python3.8
export PATH=$PYTHON_HOME/bin:$PATH
保存退出后执行：
source ~/.bash_profile
安装完成
```

# 2、安装相关库

## 2.1 安装requests库

```shell
python3 -m pip install requests
```

## 2.2、安装rados库

需要先配置ceph源：

```shell
vim /etc/yum.repos.d/ceph.repo
```

```shell
[ceph]
name=Ceph packages for $basearch
baseurl=https://mirrors.aliyun.com/ceph/rpm-nautilus/el7/$basearch
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://download.ceph.com/keys/release.asc


[ceph-noarch]
name=Ceph noarch packages
baseurl=https://mirrors.aliyun.com/ceph/rpm-nautilus/el7/noarch
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://download.ceph.com/keys/release.asc

[ceph-source]
name=Ceph source packages
baseurl=https://mirrors.aliyun.com/ceph/rpm-nautilus/el7/SRPMS
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://download.ceph.com/keys/release.asc
```

然后执行：

```shell
yum install python3-rados -y
```

python中查询MySQL：

```python
#!/usr/bin/python3
import pymysql
# 打开数据库连接
db = pymysql.connect(host='x.x.x.x',
                     user='admin',
                     password='xxxxxx',
                     database='zabbix')

# 使用cursor()方法获取操作游标 
cursor = db.cursor()

# SQL 查询语句
sql_query = "select * from (\
       select subject subjectt, \
       SUBSTRING_INDEX(SUBSTRING_INDEX(message, 'Host: ',-1),'HostIP:',1) hostname ,\
       SUBSTRING_INDEX(from_unixtime(clock),' ',1) time_ \
       from \
       alerts) as a \
       where time_ like SUBSTRING_INDEX(sysdate(),' ',1)"
try:
   # 执行SQL语句
   cursor.execute(sql_query)
   results = cursor.fetchall()
#   print (results)
except:
   print ("Error: unable to fetch data")

# 关闭数据库连接
db.close()

# 获取所有记录列表
a_list=[]
b_list=[]
c_list=[]
d_list=[a_list, b_list, c_list] 
for row in results:
   a = row[0]
   a_list.append(a)
   b = row[1]
   b_list.append(b)
   c = row[2]
   c_list.append(c)
#   print ("a=%s,b=%s,b=%s" % (a, b, c))
#print (a_list)
#print (b_list)
#print (c_list)
print (d_list)
#输出元组，然后将元组的每个元素分别组成list，并添加到一个大的list中
```

python中从一个MySQL查询前一天的数据后将结果插入到另外一个MySQL的一张表中：

```python
#!/usr/bin/python3
import pymysql
# 打开数据库连接
db = pymysql.connect(host='xx.x.x.x',
                     user='admin',
                     password='xxxxx',
                     database='zabbix')

# 使用cursor()方法获取操作游标 
cursor = db.cursor()

# SQL 查询语句
sql_query = "select * from ( \
       select subject subjectt, SUBSTRING_INDEX(SUBSTRING_INDEX(message, 'Host: ',-1),'HostIP:',1) hostname, SUBSTRING_INDEX(from_unixtime(clock),' ',1) time_ from  alerts) \
       as a \
       where time_ like DATE_SUB(curdate(),INTERVAL 1 DAY) "
try:
   # 执行SQL语句
   cursor.execute(sql_query)
   results = cursor.fetchall()
#   list(results)
#   print (results)
except:
   print ("Error: unable to fetch data")

# 关闭数据库连接
db.close()

# 打开数据库连接
db = pymysql.connect(host='xx.x.x.xx',
                     user='root',
                     password='xxxxxx',
                     database='zabbix')

# 使用cursor()方法获取操作游标 
cursor = db.cursor()

# 获取所有记录列表
a_list=[]
for row in results:
   a = row
   a_list.append(a)
print (a_list)

# SQL 插入语句
sql_insert = "INSERT INTO alerts (subjectt, hostname, time_) values (%s, %s, %s)" 

try:
   # 执行sql语句
   cursor.executemany(sql_insert, a_list)
   # 执行sql语句
   db.commit()
except:
   # 发生错误时回滚
   db.rollback()

# 关闭数据库连接
db.close()

```

打包生成exe文件：

-i 图标路径
–icon=图标路径
-F 打包成一个exe文件
-w 使用窗口，无控制台
-c 使用控制台，无窗口
-D 创建一个目录，里面包含exe以及其他一些依赖性文件

```shell
pyinstaller -F -w --icon=“C:\Download\k1.ico” 台账查询_V1.3.py
```
