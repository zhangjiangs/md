# haproxy+keepalived实现应用高可用及负载均衡

# Install

https://www.redhat.com/sysadmin/keepalived-basics
https://docs.oracle.com/cd/E37670_01/E41138/html/section_ksr_psb_nr.html
https://access.redhat.com/documentation/en-us/red_hat_cloudforms/4.6/html/high_availability_guide/configuring_haproxy
https://coding-stream-of-consciousness.com/2019/01/03/ha-proxy-centos-7-or-rhel-7-wont-bind-to-any-ports-systemd/
https://cbonte.github.io/haproxy-dconv/1.8/configuration.html

## Install Haproxy v1.8 on centos
```sh
[root@ht-haproxy-1 ~]# yum install centos-release-scl -y
[root@ht-haproxy-1 ~]# yum install rh-haproxy18-haproxy rh-haproxy18-haproxy-syspaths -y
```


```sh
[root@ht-haproxy-2 ~]# yum install centos-release-scl -y
[root@ht-haproxy-2 ~]# yum install rh-haproxy18-haproxy rh-haproxy18-haproxy-syspaths -y
```

## Install Keepalived v2.0.20 on centos
```sh
[root@ht-haproxy-1 ~]# yum install -y gcc openssl-devel wget psmisc
[root@ht-haproxy-1 ~]# wget https://www.keepalived.org/software/keepalived-2.0.20.tar.gz
[root@ht-haproxy-1 ~]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
91.121.30.175 www.keepalived.org
[root@ht-haproxy-1 ~]# tar -xf keepalived-2.0.20.tar.gz
[root@ht-haproxy-1 ~]# cd keepalived-2.0.20
[root@ht-haproxy-1 keepalived-2.0.20]# ./configure
[root@ht-haproxy-1 keepalived-2.0.20]# make
[root@ht-haproxy-1 keepalived-2.0.20]# make install
[root@ht-haproxy-1 ~]# cd ~
[root@ht-haproxy-1 ~]# keepalived --version
```


```sh
[root@ht-haproxy-2 ~]# yum install -y gcc openssl-devel wget psmisc
[root@ht-haproxy-2 ~]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
91.121.30.175 www.keepalived.org
[root@ht-haproxy-2 ~]# wget https://www.keepalived.org/software/keepalived-2.0.20.tar.gz
[root@ht-haproxy-2 ~]# tar -xf keepalived-2.0.20.tar.gz
[root@ht-haproxy-2 ~]# cd keepalived-2.0.20
[root@ht-haproxy-2 keepalived-2.0.20]# ./configure
[root@ht-haproxy-2 keepalived-2.0.20]# make
[root@ht-haproxy-2 keepalived-2.0.20]# make install
[root@ht-haproxy-2 ~]# cd ~
[root@ht-haproxy-2 ~]# keepalived --version
```

# Configure

## Configure haproxy.cfg
```sh
[root@ht-haproxy-1 ~]# vi /etc/haproxy/haproxy.cfg
```



```sh
[root@ht-haproxy-2 ~]# vi /etc/haproxy/haproxy.cfg
```

## Configure keepalived.conf
```sh
[root@ht-haproxy-1 ~]# cat /etc/keepalived/keepalived.conf
vrrp_script chk_haproxy {
script "killall -0 haproxy" # check the haproxy process
interval 2 # every 2 seconds
weight 2 # add 2 points if OK
}
vrrp_instance VI_1 {
interface eth0             # interface to monitor
state MASTER             # MASTER on haproxy1, BACKUP on haproxy2
virtual_router_id 51
priority 101             # 101 on haproxy1, 100 on haproxy2
authentication {
auth_type PASS
auth_pass 12345
}
virtual_ipaddress {
xx.xx.23.100 # virtual ip address
}
track_script {
chk_haproxy
}
}
```
```sh
[root@ht-haproxy-2 ~]# cat /etc/keepalived/keepalived.conf
vrrp_script chk_haproxy {
script "killall -0 haproxy" # check the haproxy process
interval 2 # every 2 seconds
weight 2 # add 2 points if OK
}
vrrp_instance VI_1 {
interface eth0             # interface to monitor
state BACKUP             # MASTER on haproxy1, BACKUP on haproxy2
virtual_router_id 51
priority 100            # 101 on haproxy1, 100 on haproxy2
authentication {
auth_type PASS
auth_pass 12345
}
virtual_ipaddress {
xx.xx.23.100 # virtual ip address
}
track_script {
chk_haproxy
}
}
```

## Configure sysctl.conf
```sh
[root@ht-haproxy-1 ~]# cat /etc/sysctl.conf
# System default settings live in /usr/lib/sysctl.d/00-system.conf.
# To override those settings, enter new settings here, or in an /etc/sysctl.d/<name>.conf file
#
# For more information, see sysctl.conf(5) and sysctl.d(5).
net.ipv4.ip_forward = 1
net.ipv4.ip_nonlocal_bind = 1
[root@ht-haproxy-1 ~]# sysctl -p
net.ipv4.ip_forward = 1
net.ipv4.ip_nonlocal_bind = 1
```
```sh
[root@ht-haproxy-2 ~]# cat /etc/sysctl.conf
# System default settings live in /usr/lib/sysctl.d/00-system.conf.
# To override those settings, enter new settings here, or in an /etc/sysctl.d/<name>.conf file
#
# For more information, see sysctl.conf(5) and sysctl.d(5).
net.ipv4.ip_forward = 1
net.ipv4.ip_nonlocal_bind = 1
[root@ht-haproxy-2 ~]# sysctl -p
net.ipv4.ip_forward = 1
net.ipv4.ip_nonlocal_bind = 1
```

## Configure iptables
```sh
[root@ht-haproxy-1 ~]# firewall-cmd --permanent  --add-port=22/tcp --add-port=80/tcp --add-port=443/tcp --add-port=7443/tcp  --add-port=8443/tcp  --add-port=9443/tcp && firewall-cmd --reload
[root@ht-haproxy-1 ~]# iptables -I INPUT -i eth0 -d 224.0.0.0/8 -p vrrp -j ACCEPT
[root@ht-haproxy-1 ~]# iptables -I OUTPUT -o eth0 -d 224.0.0.0/8 -p vrrp -j ACCEPT
[root@ht-haproxy-1 ~]# service iptables save
```
```sh
[root@ht-haproxy-2 ~]# firewall-cmd --permanent  --add-port=22/tcp --add-port=80/tcp --add-port=443/tcp --add-port=7443/tcp  --add-port=8443/tcp  --add-port=9443/tcp && firewall-cmd --reload
[root@ht-haproxy-2 ~]# iptables -I INPUT -i eth0 -d 224.0.0.0/8 -p vrrp -j ACCEPT
[root@ht-haproxy-2 ~]# iptables -I OUTPUT -o eth0 -d 224.0.0.0/8 -p vrrp -j ACCEPT
[root@ht-haproxy-2 ~]# service iptables save
```

## Enable haproxy log (Debugging)
```
[root@ht-haproxy-1 ~]# cat /etc/rsyslog.conf
# Provides UDP syslog reception
$ModLoad imudp
$UDPServerRun 514

local2.* /var/opt/rh/rh-haproxy18/log/haproxy.log

[root@ht-haproxy-1 ~]# service rsyslog restart
[root@ht-haproxy-1 ~]# tail -f /var/opt/rh/rh-haproxy18/log/haproxy.log
```

# Start
```sh
[root@ht-haproxy-1 ~]# setsebool -P haproxy_connect_any 1
[root@ht-haproxy-1 ~]# systemctl enable keepalived
[root@ht-haproxy-1 ~]# systemctl start keepalived
[root@ht-haproxy-1 ~]# systemctl enable haproxy
[root@ht-haproxy-1 ~]# systemctl start haproxy
```
```sh
[root@ht-haproxy-1 ~]# setsebool -P haproxy_connect_any 1
[root@ht-haproxy-2 ~]# systemctl enable keepalived
[root@ht-haproxy-2 ~]# systemctl start keepalived
[root@ht-haproxy-2 ~]# systemctl enable haproxy
[root@ht-haproxy-2 ~]# systemctl start haproxy
```

# Verify
```sh
[root@ht-haproxy-1 ~]# ip -br a
lo               UNKNOWN        127.0.0.1/8 ::1/128 
eth0             UP             xx.xx.23.21/24 xx.xx.23.100/32 fe80::cf34:131e:f89e:a212/64
```
```sh
[root@ht-haproxy-2 ~]# ip -br a
lo               UNKNOWN        127.0.0.1/8 ::1/128 
eth0             UP             xx.xx.23.22/24 fe80::cf34:131e:f89e:a212/64 fe80::11e4:de69:f358:f66c/64
```


# Refs
## cat /etc/haproxy/haproxy.cfg
```sh
[root@ht-test-1 /]# cat /etc/haproxy/haproxy.cfg 
#---------------------------------------------------------------------
# Example configuration for a possible web application.  See the
# full configuration options online.
#
#   http://haproxy.1wt.eu/download/1.8/doc/configuration.txt
#
#---------------------------------------------------------------------

#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    # to have these messages end up in /var/opt/rh/rh-haproxy18/log/haproxy.log you will
    # need to:
    #
    # 1) configure syslog to accept network log events.  This is done
    #    by adding the '-r' option to the SYSLOGD_OPTIONS in
    #    /etc/sysconfig/syslog
    #
    # 2) configure local2 events to go to the /var/opt/rh/rh-haproxy18/log/haproxy.log
    #   file. A line like the following can be added to
    #   /etc/sysconfig/syslog
    #
    #    local2.*                       /var/opt/rh/rh-haproxy18/log/haproxy.log
    #
    log         127.0.0.1 local2

    chroot      /var/opt/rh/rh-haproxy18/lib/haproxy
    pidfile     /var/run/rh-haproxy18-haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon

    # turn on stats unix socket
    stats socket /var/opt/rh/rh-haproxy18/lib/haproxy/stats

    # utilize system-wide crypto-policies
    ssl-default-bind-ciphers PROFILE=SYSTEM
    ssl-default-server-ciphers PROFILE=SYSTEM

#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000

#---------------------------------------------------------------------
# main frontend which proxys to the backends
#---------------------------------------------------------------------
frontend kube-apiserver
    mode tcp
    bind *:5000
#    acl url_static       path_beg       -i /static /images /javascript /stylesheets
#    acl url_static       path_end       -i .jpg .gif .png .css .js

#    use_backend static          if url_static
    default_backend             kube-apiserver

#---------------------------------------------------------------------
# static backend for serving up images, stylesheets and such
#---------------------------------------------------------------------
#backend static
#    balance     roundrobin
#    server      static 127.0.0.1:4331 check

#---------------------------------------------------------------------
# round robin balancing between the various backends
#---------------------------------------------------------------------
backend kube-apiserver
    mode        tcp
    balance     roundrobin
    server  ht-test-3 xx.xx.105.223:6443 check
    server  ht-test-4 xx.xx.105.224:6443 check
    server  ht-test-5 xx.xx.105.225:6443 check
#    server  app4 127.0.0.1:5004 check
```

```shell
#curl -v -k https://xx.xx.105.231:5000 测试验证haproxy的地址是否可以访问kube-apiserver服务
[root@ht-test-3 work]# curl -v -k https://xx.xx.105.231:5000
* About to connect() to xx.xx.105.231 port 5000 (#0)
*   Trying xx.xx.105.231...
* Connected to xx.xx.105.231 (xx.xx.105.231) port 5000 (#0)
* Initializing NSS with certpath: sql:/etc/pki/nssdb
* skipping SSL peer certificate verification
* NSS: client certificate not found (nickname not specified)
* SSL connection using TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
* Server certificate:
* 	subject: CN=kubernetes,OU=system,O=k8s,L=SuZhou,ST=JiangSu,C=CN
* 	start date: Aug 12 16:14:00 2021 GMT
* 	expire date: Aug 10 16:14:00 2031 GMT
* 	common name: kubernetes
* 	issuer: CN=kubernetes,OU=system,O=k8s,L=SuZhou,ST=JiangSu,C=CN
> GET / HTTP/1.1
> User-Agent: curl/7.29.0
> Host: xx.xx.105.231:5000
> Accept: */*
> 
< HTTP/1.1 401 Unauthorized
< Cache-Control: no-cache, private
< Content-Type: application/json
< Date: Fri, 13 Aug 2021 05:11:47 GMT
< Content-Length: 165
< 
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {
    
  },
  "status": "Failure",
  "message": "Unauthorized",
  "reason": "Unauthorized",
  "code": 401
* Connection #0 to host xx.xx.105.231 left intact

```

可以看到TCP/IP连接创建成功，得到响应码为401的应答。说明通过VIP地址xx.xx.105.231成功访问到了后端的kube-apiserver服务。

