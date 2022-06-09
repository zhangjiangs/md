1、创建脚本

```shell
[root@test-master-1 script]# cat port_alert.sh 
#/bin/bash
CONFIG_FILE=/script/port.conf
Check(){
    grep -vE '(^ *#|^$)' ${CONFIG_FILE} | grep -vE '^ *[0-9]+' &> /dev/null
    if [ $? -eq 0 ]
    then
        echo Error: ${CONFIG_FILE} Contains Invalid Port.
        exit 1
    else
        portarray=($(grep -vE '(^ *#|^$)' ${CONFIG_FILE} | grep -E '^ *[0-9]+'))
    fi
}
PortDiscovery(){
    length=${#portarray[@]}
    printf "{\n"
    printf  '\t'"\"data\":["
    for ((i=0;i<$length;i++))
      do
        printf '\n\t\t{'
        printf "\"{#TCP_PORT}\":\"${portarray[$i]}\"}"
        if [ $i -lt $[$length-1] ];then
                    printf ','
        fi
      done
    printf  "\n\t]\n"
    printf "}\n"
}
port(){
    Check
    PortDiscovery
}
port

```

2、配置文件：

```
[root@test-master-1 script]# cat port.conf 
6443
2379
10251
10257
2380
10252
10259
```

两个脚本，port_alert.sh为端口自发现脚本，port.conf为指定的监控端口号

