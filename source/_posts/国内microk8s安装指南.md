---
title: 国内microk8s安装指南
date: 2019-06-21 10:08:30
categories:
tags: k8s
---

microk8s 是一个专门为开发人员设计的轻量级单节点k8s包.可以用来替代minikube进行学习.

## 问题
由gfw,安装microk8s后会发现docker image无法下载的问题.(详细安装步骤参见 https://microk8s.io/#quick-start)

`microk8s.kubectl describe pods -A` 会有错误提示
 *[ERROR ImagePull]: failed to pull image [k8s.gcr.io/pause:3.1]*

## 解决办法
```bash
docker pull mirrorgooglecontainers/pause:3.1
docker pull mirrorgooglecontainers/heapster-influxdb-amd64:v1.3.3
docker pull mirrorgooglecontainers/heapster-grafana-amd64:v4.4.3
docker pull mirrorgooglecontainers/kubernetes-dashboard-amd64:v1.8.3
docker pull mirrorgooglecontainers/heapster-amd64:v1.5.2
docker pull mirrorgooglecontainers/k8s-dns-dnsmasq-nanny-amd64:1.14.7
docker pull mirrorgooglecontainers/k8s-dns-kube-dns-amd64:1.14.7
docker pull mirrorgooglecontainers/k8s-dns-sidecar-amd64:1.14.7

docker tag mirrorgooglecontainers/pause:3.1 k8s.gcr.io/pause:3.1
docker tag mirrorgooglecontainers/heapster-influxdb-amd64:v1.3.3 k8s.gcr.io/heapster-influxdb-amd64:v1.3.3
docker tag mirrorgooglecontainers/heapster-grafana-amd64:v4.4.3 k8s.gcr.io/heapster-grafana-amd64:v4.4.3
docker tag mirrorgooglecontainers/kubernetes-dashboard-amd64:v1.8.3 k8s.gcr.io/kubernetes-dashboard-amd64:v1.8.3
docker tag mirrorgooglecontainers/heapster-amd64:v1.5.2 k8s.gcr.io/heapster-amd64:v1.5.2
docker tag mirrorgooglecontainers/k8s-dns-dnsmasq-nanny-amd64:1.14.7 gcr.io/google_containers/k8s-dns-dnsmasq-nanny-amd64:1.14.7
docker tag mirrorgooglecontainers/k8s-dns-kube-dns-amd64:1.14.7 gcr.io/google_containers/k8s-dns-kube-dns-amd64:1.14.7
docker tag mirrorgooglecontainers/k8s-dns-sidecar-amd64:1.14.7 gcr.io/google_containers/k8s-dns-sidecar-amd64:1.14.7

docker save k8s.gcr.io/pause > pause.tar
docker save k8s.gcr.io/heapster-influxdb-amd64 > heapster-influxdb-amd64.tar
docker save k8s.gcr.io/heapster-grafana-amd64 > heapster-grafana-amd64.tar
docker save k8s.gcr.io/kubernetes-dashboard-amd64 > kubernetes-dashboard-amd64.tar
docker save k8s.gcr.io/heapster-amd64 > heapster-amd64.tar
docker save gcr.io/google_containers/k8s-dns-dnsmasq-nanny-amd64 > k8s-dns-dnsmasq-nanny-amd64.tar
docker save gcr.io/google_containers/k8s-dns-kube-dns-amd64 > k8s-dns-kube-dns-amd64.tar
docker save gcr.io/google_containers/k8s-dns-sidecar-amd64 > k8s-dns-sidecar-amd64.tar

microk8s.ctr -n k8s.io image import pause.tar
microk8s.ctr -n k8s.io image import heapster-influxdb-amd64.tar
microk8s.ctr -n k8s.io image import heapster-grafana-amd64.tar
microk8s.ctr -n k8s.io image import kubernetes-dashboard-amd64.tar
microk8s.ctr -n k8s.io image import heapster-amd64.tar
microk8s.ctr -n k8s.io image import k8s-dns-dnsmasq-nanny-amd64.tar
microk8s.ctr -n k8s.io image import k8s-dns-kube-dns-amd64.tar
microk8s.ctr -n k8s.io image import k8s-dns-sidecar-amd64.tar
```
然后就可以愉快的玩耍了.