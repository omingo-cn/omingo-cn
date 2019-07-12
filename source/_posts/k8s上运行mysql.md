---
title: k8s上运行mysql
date: 2019-07-12 17:19:17
categories: 
tags: k8s
---

k8s中跑mysql要求:
1. 可通过节点ip访问mysql.
2. pod可以访问mysql.
3. 机器重启后mysql数据不丢失.

```yml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: mysql
spec:
  template:
    metadata:
      labels:
        app: mysql
    spec:
      nodeName: node1 #指定pod只能调度到node1上
      containers:
      - image: mysql:5.6
        name: mysql
        env:
        - name: 'MYSQL_ROOT_PASSWORD'
          value: 'root'
        - name: "TZ"
          value: "Asia/Shanghai" #指定mysql容器的时区为CST,默认为UTC
        ports:
        - containerPort: 3306
        volumeMounts:
        - mountPath: /var/lib/mysql
          name: mysql-volume
      volumes:
      - name: mysql-volume # 使用hostPath讲数据文件挂载出来
        hostPath:
          path: /data/mysql
---
apiVersion: v1
kind: Service
metadata: 
  name: mysql
spec:
  selector:
    app: mysql
  type: NodePort
  ports:
  - port: 3306 # pod中通过 mysql:3306 访问
    nodePort: 32306 # 集群外部通过 节点IP:32306访问
```
使用hostPath类型的volume持久化数据文件,nodeName固定调度节点,NodePort暴露服务. 
肥肠地方便.