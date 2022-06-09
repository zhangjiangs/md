# Centos软件安装脚本
```shell
#!/bin/bash
# This script will install System components!

#-----------------安装wget、vim、ntp、net-tools------------------------------
echo "-----------------即将安装及配置wget、vim、ntp、net-tools--------------------------"

check_wget=`rpm -qa | grep "wget"`
check_vim=`rpm -qa | grep "vim"`
check_ntp=`rpm -qa | grep "ntp"`
check_net_tools=`rpm -qa | grep "net-tools"`
NTP=/etc/ntp.conf

if [[ ${check_wget} =~ "wget" ]] 
	then 
		echo "wget已经安装！ "
	else 
		echo "即将开始安装wget！"
		yum install -y wget 
fi

if [[ ${check_vim} =~ "vim" ]] 
	then 
		echo "vim已经安装！ "
	else 
		echo "即将开始安装vim！"
		yum install -y vim 
fi
		
if [[ ${check_ntp} =~ "ntp" ]] 
	then 
		echo "ntp已经安装！ "
	else 
		echo "即将开始安装ntp！"
		yum install -y ntp
		yum install -y ntpdate
		sed -i '17a restrict xx.xx.6.252 mask 255.255.255.0 nomodify notrap' $NTP
		sed -i '22s/^/#&/g' $NTP
		sed -i '23s/^/#&/g' $NTP
		sed -i '24s/^/#&/g' $NTP
		sed -i '25s/^/#&/g' $NTP
		sed -i '25a server xx.xx.6.252 profer' $NTP
		/usr/sbin/ntpdate xx.xx.6.252
		#echo "启动ntp服务！"
		#service ntpd start
		echo "0 12 * * * /usr/sbin/ntpdate xx.xx.6.252" >> /var/spool/cron/root
fi

if service ntpd status ;
	then
		echo "ntp服务正常启动"
	else
		echo "ntp服务异常"
fi

if [[ ${check_net_tools} =~ "net-tools" ]] 
	then 
		echo "net-tools已经安装！ "
	else 
		echo "即将开始安装net-tools！"
		yum install -y net-tools
fi		

#---------------------------------安装zabbix、filebeat------------------------
echo "-------下载zabbix-agent及filebeat的安装包--------"

if [ ! -d "/download" ]
    then
        mkdir -p /download
    else
        echo "download文件夹已经存在。"
fi

if [ ! -f "/download/zabbix-agent-4.0.15-1.el7.x86_64.rpm" ]
	then
		wget ftp://xx.xx.61.12/zabbix-agent-4.0.15-1.el7.x86_64.rpm --ftp-user=ht --ftp-password=666666 -r --directory-prefix=/download -nH
	else
		echo "zabbix-agent安装包已经存在。"
fi
		
if [ ! -f "/download/filebeat-7.6.0-x86_64.rpm" ]
	then
		wget ftp://xx.xx.61.12/filebeat-7.6.0-x86_64.rpm --ftp-user=ht --ftp-password=666666 -r --directory-prefix=/download -nH
	else
		echo "filebeat安装包已经存在。"
fi

#---------------安装filebeat-------------------
echo "--------------即将开始安装filebeat----------------"

filebeat_path=/etc/filebeat/filebeat.yml
check_filebeat=`rpm -qa | grep "filebeat"`

if [[ $check_filebeat =~ "filebeat" ]] 
	then 
		echo "filebeat 已经安装！ "
	else 
		echo "即将开始安装filebeat！"
		cd /download || exit
		yum -y install filebeat-7.6.0-x86_64.rpm
		sed -i '24d' $filebeat_path
		sed -i '25a \ \ enabled: true' $filebeat_path
		sed -i '28s/*.log/messages/g' $filebeat_path
		sed -i '29a \ \ fields:' $filebeat_path
		sed -i '30a \ \ \ \ log_source: linux' $filebeat_path
		sed -i '150s/^/#&/g' $filebeat_path
		sed -i '152s/^/#&/g' $filebeat_path
		sed -i '177a #--------------------- redis output ----------------------' $filebeat_path
		sed -i '178a output.redis:' $filebeat_path
		sed -i '179a \ \ hosts: ["xx.xx.2.56:6379"]' $filebeat_path
		sed -i '180a \ \ password: "123456"' $filebeat_path
		sed -i '181a \ \ key: "linux-log"' $filebeat_path
		sed -i '182a \ \ db: 10' $filebeat_path
		sed -i '183a \ \ timeout: 5' $filebeat_path

		service filebeat start				
fi

if service filebeat status
    then
	    echo "filebeat服务正常启动"
    else
	    echo "filebeat服务异常"
fi

#-----------------安装zabbix------------------------
echo "----------即将开始安装zabbix-agent-------------"

host=$(echo `hostname` | tr '[A-Z]' '[a-z]')
zabbix_path=/etc/zabbix/zabbix_agentd.conf
check_zabbix=`rpm -qa | grep "zabbix"`

if [[ $check_zabbix =~ "zabbix" ]] 
	then 
		echo "zabbix已经安装！ "
	else 
		echo "即将开始安装zabbix！"
		cd /download || exit
		yum -y install zabbix-agent-4.0.15-1.el7.x86_64.rpm
		sed -i '73a EnableRemoteCommands=1' $zabbix_path
		sed -i '99s/127.0.0.1/xx.xx.2.4/g' $zabbix_path
		sed -i '140s/127.0.0.1/xx.xx.2.4/g' $zabbix_path
		sed -i "151s/Zabbix server/$host/g" $zabbix_path
		
		service zabbix-agent start
fi

if service zabbix-agent status
	then
		echo "zabbix-agent服务正常启动"
	else
		echo "zabbix-agent服务异常"
fi

chkconfig filebeat on
chkconfig zabbix-agent on

```

# 系统配置点检脚本

## 1、获取系统配置：

```sh
#!/bin/bash
#This script is used to check the server 

HOST=$(echo `hostname`)
#IP=`/sbin/ifconfig -a|grep inet|grep -v 127.0.0.1|grep -v inet6|awk '{print $2}'|tr -d "addr:"|sed -n 2p`
IP=`/sbin/ifconfig -a|grep inet|grep -v 127.0.0.1|grep -v inet6|awk '{print $2}'|tr -d "addr:"|grep xx.xx.`
TODAY=`date +%Y%m%d`
TODAYSYSCHECKFILE=/system_check_file/${HOST}_${IP}_system_check_${TODAY}.txt

#system info 
system_info() { 
echo "**********************************************" 
echo "system info:" 
echo 
echo "   System-release : `cat /etc/redhat-release`" 
echo "   Kernel-release : `uname -a|awk '{print $1,$3}'`" 
echo "   Server-Model : `dmidecode | grep "Product Name:"|sed -n '1p'|awk -F': ' '{print $2}'`" 
echo 
} 
 
#CPU info 
cpu_info() { 
echo "**********************************************" 
echo "CPU info:" 
echo 
echo "    Frequency : `cat /proc/cpuinfo | grep "model name" | uniq |awk -F': ' '{print $2}'`" 
echo "    CPU cores:  `cat /proc/cpuinfo | grep "cpu cores" | uniq |awk -F': ' '{print $2}'`" 
echo "    Logic Count : `cat /proc/cpuinfo | grep "processor" | sort -u| wc -l `" 
echo "    Physical Count : `cat /proc/cpuinfo | grep "physical id" | sort -u| wc -l`" 
echo "    Cache size : `cat /proc/cpuinfo| grep "cache size"|uniq|awk '{print $4,$5}'`" 
echo 
} 
 
#memory info 
mem_info() { 
memory=`dmidecode |grep "Range Size"|head -1|awk '{print $3$4}'` 
mem_size=`echo "This server has ${memory} memory."` 
 
echo "**********************************************" 
echo "Memory info:" 
echo 
echo "   Total : ${mem_size}" 
echo "   Count : `dmidecode |grep -A16 "Memory Device$"|grep Size|awk '{if($2!~/No/) print $0}'|wc -l`" 
dmidecode |grep -A20 "Memory Device$"|grep Size|sed '{s/^       */   /g};{/No/d}' 
echo 
} 


#disk and partitions 
swap_pos=`cat /proc/swaps|sed -n '2p'|awk '{print $1}'` 
disk_info() { 
echo "**********************************************" 
echo "Hard disk info:" 
echo 
echo "`fdisk -l|grep Disk|awk -F, '{print $1}'`" 
} 


#software package 
software_info() { 
echo "**********************************************" 
echo "SELinux is `cat /etc/selinux/config`" 
} 
 
#error()
#{
#    echo "$1" 1>&2
#    exit 1
#}

if [ ! -d "/system_check_file" ];
        then
                mkdir -p /system_check_file
fi

system_info > ${TODAYSYSCHECKFILE} 2>&1
cpu_info >> ${TODAYSYSCHECKFILE} 2>&1
mem_info >> ${TODAYSYSCHECKFILE} 2>&1
disk_info >> ${TODAYSYSCHECKFILE} 2>&1
```

## 2、和前一天的系统配置做对比并发送邮件

```sh
#!/bin/bash
#ifconfig -a | awk '{print $2}' | grep xx.xx.
#IP=`/sbin/ifconfig -a|grep inet|grep -v 127.0.0.1|grep -v inet6|awk '{print $2}'|tr -d "addr:"|sed -n 2p`
IP=`/sbin/ifconfig -a|grep inet|grep -v 127.0.0.1|grep -v inet6|awk '{print $2}'|tr -d "addr:"|grep xx.xx.`
TODAY=`date +%Y%m%d`

YESTERDAY=`date -d"yesterday" +%Y%m%d`

HOST=$(echo `hostname`)

TODAYSYSCHECKFILE=/system_check_file/${HOST}_${IP}_system_check_${TODAY}.txt

YESTERDAYCHECKFILE=/system_check_file/${HOST}_${IP}_system_check_${YESTERDAY}.txt

MAIL="zhangjs@xxxx.com"

DAYS=21

mailconf=/etc/mail.rc

check_results=`rpm -qa | grep "mailx"`

find /system_check_file/ -mtime +${DAYS} -delete

if [[ $check_results =~ "mailx" ]] 
	then 		
		echo "邮件组件已经安装，检查是否已经进行配置："
		check_config=`cat /etc/mail.rc | grep "dbcheck@xxxx.com"`
		if [[ $check_config =~ "dbcheck@xxxx.com" ]] 
			then echo "邮件组件已经配置。"
		else
			echo "邮件组件没有配置，下面进行配置" 
			echo "set from=dbcheck@xxxx.com" | tee -a $mailconf 
		    echo "set smtp=mail.xxxx.com" | tee -a $mailconf
        	echo "set smtp-auth-user=dbcheck@xxxx.com" | tee -a $mailconf
        	echo "set smtp-auth-password=xxxxx@12345" | tee -a $mailconf
        	echo "set smtp-auth=login" | tee -a $mailconf
		fi
else
	echo "邮件组件未安装，现在开始安装："
	yum -y install mailx
	echo "安装完成！开始配置邮件发送设置："
	echo "set from=dbcheck@xxxx.com" | tee -a $mailconf
	echo "set smtp=mail.xxxx.com" | tee -a $mailconf
	echo "set smtp-auth-user=dbcheck@xxxx.com" | tee -a $mailconf
	echo "set smtp-auth-password=xxxxx@12345" | tee -a $mailconf
	echo "set smtp-auth=login" | tee -a $mailconf
fi

if [ ! -f "${YESTERDAYCHECKFILE}" ]
	then
		DIFFERENT=`diff ${TODAYSYSCHECKFILE} ${YESTERDAYCHECKFILE}`
		`echo "前一天的文件不存在" | mail -s "${HOST}_${IP}_服务器硬件校验信息异常。" ${MAIL}` && echo "前一天的文件不存在，已经发送告警邮件。"
	elif [ ! -n "${DIFFERENT}" ]
		then
            echo "服务器硬件未出现变更。"
        else
            `echo  "${DIFFERENT}" | mail -s "${HOST}_${IP}_服务器硬件出现变更。" ${MAIL}` && echo "服务器出现硬件变更，已经发送告警邮件。"
fi

```

# 巡检点检相关

## 1、自动化点检脚本

```sh
#!/bin/bash
echo "----------------Oracle数据库点检结果--------------------" 
echo "" 
echo "检查日期：`date`"
echo ""

echo "1、操作系统CPU空闲时间"
CPUID=`vmstat | sed -n 3p | awk '{print $15}'`
echo "cpu_id = $CPUID% "
echo ""

echo "2、操作系统内存使用情况"
free -m
MemoryUsage=`free -m | sed -n 2p|awk '{print $3/$2*100"%"}'`
echo "Memory usage = $MemoryUsage"
echo ""

echo "3、操作系统磁盘使用情况"
df -h
echo ""

echo "4、数据库备份情况"
ls -lh /db_backup
echo ""

echo "5、检查Oracle监听状态"
su - oracle << EOF
lsnrctl status;
exit;
EOF
echo ""

echo "6、Oracle运行状态"
SQL="select name,open_mode from v\$database;"
su - oracle << EOF
sqlplus / as sysdba;
$SQL

set linesize 200;
SELECT
	SUM( bytes ) / ( 1024 * 1024 ) AS free_space,
	tablespace_name 
FROM
	dba_free_space 
GROUP BY
	tablespace_name;
SELECT
	a.tablespace_name,
	a.bytes total,
	b.bytes used,
	c.bytes free,
	( b.bytes * 100 ) / a.bytes "% USED ",
	( c.bytes * 100 ) / a.bytes "% FREE " 
FROM
	sys.sm\$ts_avail a,
	sys.sm\$ts_used b,
	sys.sm\$ts_free c 
WHERE
	a.tablespace_name = b.tablespace_name 
	AND a.tablespace_name = c.tablespace_name;

select tablespace_name,file_id,autoextensible from dba_data_files;
exit;
EOF
```

## 2、邮件发送脚本

```sh
#!/bin/bash
IP=`/sbin/ifconfig -a|grep inet|grep -v 127.0.0.1|grep -v inet6|awk '{print $2}'|tr -d "addr:"`

TODAY=`date +%Y%m%d%H%M`

/scripts/oracle.sh > /scripts/oracle_check_file/gd_paichan_${IP}_dbcheck_${TODAY}.txt 2>&1

DBCHECKFILE=`cat /scripts/oracle_check_file/gd_paichan_${IP}_dbcheck_${TODAY}.txt`

echo "$DBCHECKFILE" | mail -s "数据库点检" -a /scripts/oracle_check_file/gd_paichan_${IP}_dbcheck_${TODAY}.txt zhangjs@xxxx.com
```

# mongodb备份脚本

```sh
#!/bin/bash
#backup MongoDB

#mongodump命令路径
DUMP=/usr/bin/mongodump
#备份存放路径
BACKUP=/mongodb/bak
#获取当前系统时间
DATE=`date +%Y%m%d`
#数据库账号
DB_USER=aof-rw
#数据库密码
DB_PWD=aof_2017
#数据库名称
DB_NAME=HTGQ_WMS

#备份全部数据库
$DUMP --port 28017 -u $DB_USER -p $DB_PWD -d $DB_NAME -o $BACKUP/$DATE
```

# MySQL自动化备份脚本

## 全备

```sh
#!/bin/sh
 
INNOBACKUPEX=innobackupex
INNOBACKUPEXFULL=/usr/bin/$INNOBACKUPEX
TODAY=`date +%Y%m%d%H%M`
YESTERDAY=`date -d"yesterday" +%Y%m%d%H%M`
USEROPTIONS="--user=admin --password=admin"
TMPFILE="/home/databackup/logs/innobackup_$TODAY.$$.tmp"
MYCNF=/etc/my.cnf
MYSQL=/usr/local/mysql/bin/mysql
BACKUPDIR=/home/mysql_backup # 备份的主目录
FULLBACKUPDIR=/home/databackup/full # 全库备份的目录
INCRBACKUPDIR=/home/databackup/incr # 增量备份的目录

 
# Grab start time
#############################################################################
# Display error message and exit
#############################################################################
error()
{
    echo "$1" 1>&2
    exit 1
}

 # Check options before proceeding
if [ ! -x $INNOBACKUPEXFULL ]; then
  error "$INNOBACKUPEXFULL does not exist."
fi
 
if [ ! -d $BACKUPDIR ]; then
  error "Backup destination folder: $BACKUPDIR does not exist."
fi

# Some info output
echo "----------------------------"
echo
echo "$0: MySQL backup script"
echo "started: `date`"
echo

 # Create full and incr backup directories if they not exist.
for i in $FULLBACKUPDIR $INCRBACKUPDIR
do
        if [ ! -d $i ]; then
                mkdir -pv $i
        fi
done

echo "压缩前一天的备份"
cd /home/databackup
tar -zcvf $BACKUPDIR/$YESTERDAY.tar.gz ./full/ ./incr/

if [ $? = 0 ]; then
  rm -rf /home/databackup/full /home/databackup/incr
  echo "Running new full backup."
  innobackupex  --defaults-file=$MYCNF $USEROPTIONS --parallel=16 --compress --compress-threads=8  $FULLBACKUPDIR > $TMPFILE 2>&1
else
  echo "Error with tar."
fi
 
if [ -z "`tail -1 $TMPFILE | grep 'completed OK!'`" ] ; then
 echo "$INNOBACKUPEX failed:"; echo
 echo "---------- ERROR OUTPUT from $INNOBACKUPEX ----------"
 cat $TMPFILE
 exit 1
fi
 
# 这里获取这次备份的目录 
THISBACKUP=`awk -- "/Backup created in directory/ { split( \\\$0, p, \"'\" ) ; print p[2] }" $TMPFILE`
echo "THISBACKUP=$THISBACKUP"


echo "Databases backed up successfully to: $THISBACKUP"
 
# Cleanup
echo "delete tar files of 3 days ago"
find $BACKUPDIR/ -mtime +2 -name "*.tar.gz"  -exec rm -rf {} \;
 
echo
echo "completed: `date`"
exit 0

```

## 增备

```sh
#!/bin/sh

INNOBACKUPEX=innobackupex
INNOBACKUPEXFULL=/usr/bin/$INNOBACKUPEX
TODAY=`date +%Y%m%d%H%M`
USEROPTIONS="--user=admin --password=admin"
TMPFILE="/home/databackup/logs/incr_$TODAY.$$.tmp"
MYCNF=/etc/my.cnf
MYSQL=/usr/local/mysql/bin/mysql
BACKUPDIR=/home/mysql_backup # 备份的主目录
FULLBACKUPDIR=/home/databackup/full # 全库备份的目录
INCRBACKUPDIR=/home/databackup/incr # 增量备份的目录
 
#############################################################################
# Display error message and exit
#############################################################################
error()
{
    echo "$1" 1>&2
    exit 1
}
 
# Check options before proceeding
if [ ! -x $INNOBACKUPEXFULL ]; then
  error "$INNOBACKUPEXFULL does not exist."
fi
 
if [ ! -d $BACKUPDIR ]; then
  error "Backup destination folder: $BACKUPDIR does not exist."
fi
 
# Some info output
echo "----------------------------"
echo
echo "$0: MySQL backup script"
echo "started: `date`"
echo
 
# Create full and incr backup directories if they not exist.
for i in $FULLBACKUPDIR $INCRBACKUPDIR
do
        if [ ! -d $i ]; then
                mkdir -pv $i
        fi
done
 
# Find latest full backup
LATEST_FULL=`find $FULLBACKUPDIR -mindepth 1 -maxdepth 1 -type d -printf "%P\n"`
echo "LATEST_FULL=$LATEST_FULL" 
 
# Run an incremental backup if latest full is still valid.
# Create incremental backups dir if not exists.
TMPINCRDIR=$INCRBACKUPDIR/$LATEST_FULL
mkdir -p $TMPINCRDIR
BACKTYPE="incr"
# Find latest incremental backup.
LATEST_INCR=`find $TMPINCRDIR -mindepth 1 -maxdepth 1 -type d | sort -nr | head -1`
echo "LATEST_INCR=$LATEST_INCR"
  # If this is the first incremental, use the full as base. Otherwise, use the latest incremental as base.
if [ ! $LATEST_INCR ] ; then
  INCRBASEDIR=$FULLBACKUPDIR/$LATEST_FULL
else
  INCRBASEDIR=$LATEST_INCR
fi
echo "Running new incremental backup using $INCRBASEDIR as base."
innobackupex --defaults-file=$MYCNF $USEROPTIONS --parallel=16 --compress --compress-threads=8      --incremental $TMPINCRDIR --incremental-basedir $INCRBASEDIR > $TMPFILE 2>&1


if [ -z "`tail -1 $TMPFILE | grep 'completed OK!'`" ] ; then
 echo "$INNOBACKUPEX failed:"; echo
 echo "---------- ERROR OUTPUT from $INNOBACKUPEX ----------"
 exit 1
fi
 
# 这里获取这次备份的目录 
THISBACKUP=`awk -- "/Backup created in directory/ { split( \\\$0, p, \"'\" ) ; print p[2] }" $TMPFILE`
echo "THISBACKUP=$THISBACKUP"
echo
echo "Databases backed up successfully to: $THISBACKUP"
 
echo
echo "incremental completed: `date`"
exit 0
```

