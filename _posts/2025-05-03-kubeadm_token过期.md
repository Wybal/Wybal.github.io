---
title: kubeadm_token过期
date: 2025-05-03 22:36:00 +0800
categories: [k8s,kubeadm_token过期]
tags: [k8s,kubeadm_token过期]
---

**默认情况下，令牌会在 24 小时后过期。如果要在当前令牌过期后将节点加入集群， 则可以通过在控制平面节点上运行以下命令来创建新令牌**
##### 一、分别获取token和ca-cert-hash的值
1. 如果没有令牌，可以通过在控制平面节点上运行以下命令来获取令牌
`kubeadm token create`
```
[root@master ~]# kubeadm token create
md76nd.39rycar2x8nj697x
```
2. 如果没有 --discovery-token-ca-cert-hash 的值，则可以通过在控制平面节点上执行以下命令链来获取它
`openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'`
```
[root@master ~]# openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
05fff7ddf0b5a6b9d4a7fd40de6d926019190773e8a067462625314d3548c987
```
3. 查看token
`kubeadm token list`
```
[root@master ~]# kubeadm token list
TOKEN                     TTL  EXPIRES                USAGES                   DESCRIPTION  EXTRA GROUPS
md76nd.39rycar2x8nj697x   23h  2024-01-12T02:05:38Z   authentication,signing   <none>       system:bootstrappers:kubeadm:default-node-token
```


##### 二、直接打印join的命令
一般不用上面的分别获取 token 和 ca-cert-hash 方式，可以执行以下命令一气呵成：
`kubeadm token create --print-join-command`
```
[root@master ~]# kubeadm token create --print-join-command
kubeadm join cluster-endpoint:6443 --token 2k9wqo.o8hsb9lipavh7dm5 --discovery-token-ca-cert-hash sha256:05fff7ddf0b5a6b9d4a7fd40de6d926019190773e8a067462625314d3548c987 
```
如果是需要加入master节点在kubeadm join命令最后跟上--control-plane 
加入control-plane节点前，需要先拷贝control-plane相关ca证书再join集群

>由于集群节点通常是按顺序初始化的，CoreDNS Pod 很可能都运行在第一个控制面节点上。 为了提供更高的可用性，请在加入至少一个新节点后 使用 kubectl -n kube-system rollout restart deployment coredns 命令，重新平衡这些 CoreDNS Pod。 