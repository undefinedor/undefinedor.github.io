---
layout: post
title:  "Kubernetes:Kubeadm"
date:   2021-12-28 20:30:00 +0800
categories: Golang
---

> 趁着最近成为无业游民的时间，花费一些时间来整理K8S相关的知识

# Kubeadm
Kubeadm是一个专门用于创建K8SCluster的工具，最重要的两个命令分别是`kubeadm init`与`kubeadm join`两个命令，下面我们会就这两个最重要的命令进行分析。

## kubeadm init
众所周知，K8S集群通常是一个或多个**control plane**和多个**work node**组成。control plane可以类比成人类的大脑一样指挥work node进行工作。
![pic](https://kubernetes.io/images/docs/kubernetes-cluster-architecture.svg)
如图所示：control plane主要是由**kube-apiserver**,**etcd**,**kube-scheduler**,**kube-controller-manager**构成。而kubeadm init就是创建control plane的。

## kubeadm init步骤解析
kubeadm init的执行会拆分成以下这些步骤

- preflight:主要对CPU，内存，端口，文件路径进行检查
- certs:创建并维护证书
- kubeconfig:生成`admin.conf`,`kubelet.conf`等用于建立control plane的所有kubeconfig文件和admin kubeconfig文件
- etcd: 生成本地etcd静态pod配置文件
- control-plane:生成用于建立control plane的所有必要的静态Pod配置文件。
- kubelet-start:写入kubelet配置并启动/重启kubelet。在这个阶段，kubelet会启动属于control plane组件的静态Pod。
- wait-control-plane:这个阶段会等待control plane的完全启动
- upload-config: 将kubeadm与kubelet配置作为一个ConfigMap并上传到K8S集群
- upload-certs:将证书作为Secret上传到K8S集群
- mark-control-plane: 标记节点为控制平面节点
- bootstrap-token:生成用于将节点加入集群的引导令牌
- kubelet-finalize:TLS 引导后更新与 kubelet 相关的设置
- addon: 安装必要的插件以通过一致性测试
- show-join-command: 在control plane与work node上显示join命令
