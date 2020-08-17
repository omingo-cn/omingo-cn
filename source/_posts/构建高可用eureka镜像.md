---
title: 构建高可用eureka镜像
date: 2020-08-17 13:46:19
categories:  微服务
tags: 
- 服务发现
- 高可用
---

服务注册和发现组件是微服务架构中很重要的一个组件，在生产环境中要避免产生单点故障。一定要跑多个互为备份的实例。本文讲述如何构建一个在k8s环境中可自由扩展实例的eureka镜像。

## 应用配置
如何创建`spring-cloud`eureka项目就不多数了，我只列一下`application.yaml`中的关键配置。

```yaml
server:
  port: 8761
management:
  server:
    port: 8081
  endpoints:
    web:
      exposure:
        include: '*'
  endpoint:
    health:
      show-details: always
```
## Dockfile 
`Dockerfile`是打包的关键，采用`openjdk:8-jre-alpine`作为基础镜像。
```dockerfile
FROM openjdk:8-jre-alpine
MAINTAINER wujiming <wzslw@163.com>

#  安装curl 和 bash
RUN apk add --update \ 
    curl bash \
    && rm -rf /var/cache/apk/*

# 解决容器内时区问题
ENV TZ=Asia/Shanghai

COPY target/*.jar app.jar
COPY entrypoint.sh /entrypoint.sh  
RUN chmod +x /entrypoint.sh
RUN echo $(date) > /image_built_at

ENTRYPOINT ["/entrypoint.sh"]
CMD ["java","-jar","/app.jar", "-Djava.security.egd=file:/dev/./urandom"]

EXPOSE 8081 8761
```

entrypoint.sh
```bash
#!/usr/bin/env bash
set -e

MY_POD_NAME="mypodname"
MY_IN_SERVICE_NAME="myinservicename"
MY_POD_NAMESPACE="mypodnamespace"

EUREKA_HOST_NAME="$MY_POD_NAME.$MY_IN_SERVICE_NAME.$MY_POD_NAMESPACE"
export EUREKA_HOST_NAME=$EUREKA_HOST_NAME

BOOL_REGISTER="true"
BOOL_FETCH="true"
EUREKA_REPLICAS=3
EUREKA_REPLICAS=${EUREKA_REPLICAS:-"1"} # 默认副本数为1

if [ $EUREKA_REPLICAS = 1 ];then
  echo "Eureka副本数为1."
  BOOL_REGISTER="false"
  BOOL_FETCH="false"
  EUREKA_URL_LIST="http://localhost:8761/eureka/,"
  echo "EUREKA_URL_LIST is $EUREKA_URL_LIST"
else
  echo "Eureka副本数为 $EUREKA_REPLICAS"
  BOOL_REGISTER="true"
	BOOL_FETCH="true"
	for ((i=0 ; i<$EUREKA_REPLICAS ; i++))
	do
	  temp="http://$MY_POD_NAME-$i.$MY_IN_SERVICE_NAME.$MY_POD_NAMESPACE:8761/eureka/,"
	  EUREKA_URL_LIST="$EUREKA_URL_LIST$temp"
	done
fi

EUREKA_URL_LIST=${EUREKA_URL_LIST%?}
echo "EUREKA_URL_LIST is $EUREKA_URL_LIST"

export EUREKA_CLIENT_SERVICEURL_DEFAULTZONE=$EUREKA_URL_LIST
export EUREKA_CLIENT_REGISTERWITHEUREKA=$BOOL_REGISTER
export EUREKA_CLIENT_FETCHREGISTRY=$BOOL_FETCH

exec "$@"
```
将`Dockerfile`和`entrypoint.sh`放到项目根目录下，应用程序打包后，构建docker镜像就可以了。

## 服务编排
上面打包好的镜像需要部署到k8s中,。
```yaml
apiVersion: v1
kind: Service
metadata:
  name: eureka
  labels:
    app: eureka
spec:
  ports:
    - port: 8761
      name: eureka
  clusterIP: None
  selector:
    app: eureka
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: eureka-server
spec:
  podManagementPolicy: Parallel # 并行启动
  replicas: 2 # 副本数
  selector:
    matchLabels:
      app: eureka
  serviceName: eureka-server
  template:
    metadata:
      labels:
        app: eureka
    spec:
      containers:
      - env:
        - name: EUREKA_REPLICAS
          value: "2" #跟副本数保持一致
        - name: MY_IN_SERVICE_NAME
          value: eureka-server #跟上面定义的Headless Service名字保持一致
        - name: MY_POD_NAME
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.name
        - name: MY_POD_NAMESPACE
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.namespace
        image: eureka-server # 和你自己的镜像名字保持一致
        livenessProbe:
          failureThreshold: 3
          httpGet:
            path: /actuator/health
            port: 8081 # 与应用配置中的management.server.port保持一致
            scheme: HTTP
          initialDelaySeconds: 10
          periodSeconds: 2
          successThreshold: 1
          timeoutSeconds: 2
        name: eureka-server
        ports:
        - containerPort: 8761
          name: tcp8761
          protocol: TCP
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: /actuator/health
            port: 8081 # 与应用配置中的management.server.port保持一致
            scheme: HTTP
          initialDelaySeconds: 10
          periodSeconds: 2
          successThreshold: 2
          timeoutSeconds: 2
        resources:
          limits:
            memory: 1Gi
          requests:
            memory: 512Mi
```
大功告成。