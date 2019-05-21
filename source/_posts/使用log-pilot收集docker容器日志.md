---
title: 使用log-pilot收集docker容器日志
date: 2019-05-21 16:25:15
categories:
tags:
 - 日志
---

本文档介绍一款新的 Docker 日志收集工具：log-pilot。log-pilot 是阿里云提供的日志收集镜像。我们可以在每台机器上部署一个 log-pilot 实例，就可以收集机器上所有 Docker 应用日志。(注意：只支持Linux版本的Docker，不支持Windows/Mac版)。

log-pilot 具有如下特性：

* 一个单独的 log 进程收集机器上所有容器的日志。不需要为每个容器启动一个 log 进程。
* 支持文件日志和 stdout。docker log dirver 亦或 logspout 只能处理 stdout，log-pilot 不仅支持收集 stdout 日志，还可以收集文件日志。
* 声明式配置。当您的容器有日志要收集，只要通过 label 声明要收集的日志文件的路径，无需改动其他任何配置，log-pilot 就会自动收集新容器的日志。
* 支持多种日志存储方式。无论是强大的阿里云日志服务，还是比较流行的 elasticsearch 组合，甚至是 graylog，log-pilot 都能把日志投递到正确的地点。
* 开源。log-pilot 完全开源，您可以从 Git项目地址 {% link 下载代码 https://github.com/AliyunContainerService/log-pilot %}。如果现有的功能不能满足您的需要，欢迎提 issue。

 {% asset_img 1.png %}

## 实践
先有应用使用单机docker部署,需要将docker容器产生的日志发送到kafka.

首先部署log-pilot镜像,用来感知容器日志并发送日志到目的地:

```
docker run --name log-pilot -d \
-v /var/run/docker.sock:/var/run/docker.sock \
-v /etc/localtime:/etc/localtime \
-v /:/host:ro \
--cap-add SYS_ADMIN \
-e LOGGING_OUTPUT=kafka \ #选择输入类型 kafka
-e KAFKA_BROKERS=kafka:9092 \ # 配置kafka的地址
registry.cn-hangzhou.aliyuncs.com/acs/log-pilot:0.9.5-filebeat
```
运行docker应用的时候只需要增加标签 aliyun.$name.*:
如:
```
docker run -it --rm -p 10080:8080 \
-v /usr/local/tomcat/logs \
--label aliyun.logs.catalina=stdout \ 
--label aliyun.logs.access=/usr/local/tomcat/logs/localhost_access_log.*.txt \
tomcat
```
还可以自定义输入的target:
aliyun.$name.target=&lt;target&gt;
&lt;target&gt;:自定义字符串,分别指代:
1. eleasticsearch->index
2. kafka->topic


参考文章:
{% link 日志采集利器 https://yq.aliyun.com/articles/674327 %}

