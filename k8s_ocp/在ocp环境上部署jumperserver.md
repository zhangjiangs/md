# 在Openshift上部署jumpserver堡垒机

## 1、创建名为jumperserver的project

## 2、用集群管理员账号创建名为nfsmount的service accounts，并授予挂载nfs的权限

## 3、准备镜像

需要的镜像：MySQL、redis、jumpserver

```shell
docker pull mysql
docker tag mysql registry.xxxxxxx.com/jumperserver/mysql:5.7
docker push registry.xxxxxxx.com/jumperserver/mysql:5.7

docker pull redis
docker tag redis registry.xxxxxxx.com/jumperserver/redis
docker push registry.xxxxxxx.com/jumperserver/redis

docker pull jumpserver/jms_all
docker tag jumpserver/jms_all registry.xxxxxxx.com/jumperserver/jumperserver
docker push registry.xxxxxxx.com/jumperserver/jumperserver
```

## 4、部署MySQL

### 4.1、创建deployment文件

```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: mysql
  namespace: jumperserver
  labels:
    name: mysql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: mysql
    spec:
      restartPolicy: Always
      serviceAccountName: nfsmount
      securityContext: {}
      containers:
        - resources:
            limits:
              memory: 6Gi
            requests:
              cpu: '2'
              memory: 6Gi
          terminationMessagePath: /dev/termination-log
          name: mysql
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: admin
          ports:
            - containerPort: 3306
              protocol: TCP
          imagePullPolicy: Always
          volumeMounts:
            - name: nfs-volume
              mountPath: /var/lib/mysql
              subPath: jumperserver/data
          image: registry.xxxxxxx.com/jumperserver/mysql
      serviceAccount: nfsmount
      volumes:
        - name: nfs-volume
          nfs:
            server: xx.xx.23.149
            path: /nfs
```

### 4.2、进入MySQL容器修改相关配置

```sql
mysql -u root -p 
输入密码admin

mysql> create user'admin'@'%'identified by'admin';
Query OK, 0 rows affected (0.02 sec)

mysql> CREATE DATABASE IF NOT EXISTS jumperserver DEFAULT CHARSET utf8 COLLATE utf8_general_ci;
Query OK, 1 row affected, 2 warnings (0.00 sec)

mysql>  grant create,alter,drop,select,insert,update,delete on jumperserver.* to admin@'%';   
Query OK, 0 rows affected (0.00 sec)

mysql>  grant all privileges on *.* to 'admin'@'%' identified by 'admin' with grant option;

mysql> flush privileges; 
Query OK, 0 rows affected (0.01 sec)
```

4.3、创建service

```yaml
kind: Service
apiVersion: v1
metadata:
  name: mysql
  namespace: jumperserver
spec:
  ports:
    - protocol: TCP
      port: 3306
      targetPort: 3306
      nodePort: 31060
  selector:
    app: mysql
  type: NodePort
```

## 5、部署redis

### 5.1、创建configmap

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: redis-config
  namespace: jumperserver
  labels:
    app: redis
data:
  redis.conf: |-
    dir /usr/local/etc/redis/data
    port 6379
    bind 0.0.0.0
    appendonly yes
    protected-mode no
    requirepass 123456
    pidfile /usr/local/etc/redis/redis-6379.pid
```

### 5.2、创建deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: jumperserver
  labels:
    app: redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: registry.xxxxxxx.com/jumperserver/redis
          command:
            - "sh"
            - "-c"
            - "redis-server /usr/local/etc/redis/redis.conf"
          ports:
            - containerPort: 6379
          resources:
            limits:
              cpu: 1000m
              memory: 1024Mi
            requests:
              cpu: 1000m
              memory: 1024Mi
          livenessProbe:
            tcpSocket:
              port: 6379
            initialDelaySeconds: 300
            timeoutSeconds: 1
            periodSeconds: 10
            successThreshold: 1
            failureThreshold: 3
          readinessProbe:
            tcpSocket:
              port: 6379
            initialDelaySeconds: 5
            timeoutSeconds: 1
            periodSeconds: 10
            successThreshold: 1
            failureThreshold: 3
          volumeMounts:
            - name: nfs-volume
              mountPath: /usr/local/etc/redis/data
              subPath: jumperserver/redis
            - name: config
              mountPath: /usr/local/etc/redis/redis.conf
              subPath: jumperserver/redis/redis.conf
      serviceAccount: nfsmount
      volumes:
        - name: nfs-volume
          nfs:
            server: xx.xx.23.149
            path: /nfs
        - name: config
          configMap:
            name: redis-config
```

### 5.3、创建service

```yaml
kind: Service
apiVersion: v1
metadata:
  name: redis
  namespace: jumperserver
spec:
  ports:
    - protocol: TCP
      port: 6379
      targetPort: 6379
      nodePort: 32379
  selector:
    app: redis
  type: NodePort
```

## 6、部署jumpserver

### 6.1、准备jumperserver镜像

```
docker pull jumpserver/jms_all
```

### 6.2、部署deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jumpserver
  namespace: jumperserver
spec:
  selector:
    matchLabels:
      app: jumpserver
  replicas: 3
  template:
    metadata:
      labels:
        app: jumpserver
    spec:
      containers:
        - name: jumpserver
          image: registry.xxxxxxx.com/jumperserver/jumperserver
          imagePullPolicy: IfNotPresent
          env:
          - name: SECRET_KEY
            value: "Dpdjj6OtYgllQz3g20WV4WZwG2k1U25whSWJHR29rYbQdEREPk"
          - name: BOOTSTRAP_TOKEN
            value: "VUyQLAigVekFtJFU"
          - name: DB_ENGINE
            value: "mysql"
          - name: DB_HOST
            value: "10.60.54.229"
          - name: DB_PORT
            value: "3306"
          - name: DB_USER
            value: "admin"
          - name: "DB_PASSWORD"
            value: "admin"
          - name: DB_NAME
            value: "jumperserver"
          - name: REDIS_HOST
            value: "10.60.171.21"
          - name: REDIS_PASSWORD
            value: "1"
          - name: REDIS_PORT
            value: "6379"
          ports:
            - containerPort: 80
              name: http
              protocol: TCP
            - containerPort: 22
              name: ssh
              protocol: TCP
          volumeMounts:
            - mountPath: /opt/jumpserver/data/media
              name: nfs-volume
              subPath: jumperserver/jumperserver
      serviceAccount: nfsmount
      volumes:
        - name: nfs-volume
          nfs:
            server: xx.xx.23.149
            path: /nfs
```

```
1.将相应的环境变量的值替换成自己的
2.SECRET_KEY和BOOTSTRAP_TOKEN的值可以通过jumpserver官网给的脚步生成
3.数据库和redis的密码不要使用特殊符号，使用特殊符号在初始化的时候配置文件回不正常，导致初始化失败
```

### 6.3、部署service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: jumperserver
  namespace: jumperserver
spec:
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 80
    - name: ssh
      protocol: TCP
      port: 2222
      targetPort: 2222
  selector:
    app: jumpserver
  clusterIP: 10.60.42.228
  type: ClusterIP
  sessionAffinity: None
```

### 6.4、部署route

http://jumpserver.apps.dev.xxxxxxx.com/

