---
title: "CoreDNS生产案：pod出现dns解析大量失败的问题"
date: 2020-03-31T17:26:43+08:00
draft: true
categories: ["k8s"]
keywords: ["coredns"]  
description: "CoreDNS生产案：pod出现dns解析大量失败的问题" 
summaryLength: 100
draft: false
comment: true
toc: true
autoCollapseToc: true
postMetaInFooter: false
hiddenFromHomePage: false
contentCopyright: true
reward: true
mathjax: true
mathjaxEnableSingleDollar: false
mathjaxEnableAutoNumber: false
---

### 问题

收到阿里云K8S集群监控告警：

CoreDNS 5分钟内NXDOMAIN响应百分比大于50% k8s.coredns.response[aliyun,172.22.82.25:9153,NXDOMAIN]

是coredns的172.22.82.25的这个pod出现dns解析大量失败的问题。（解析成功的日志是NOERROR，解析失败是：NXDMAIN）

### 分析

首先是查看这个coredns pod的日志：

```
[root@iZbp16er8wkobo2a165ekzZ ~]# kubectl  get pods -n kube-system -o wide | grep coredns |grep 172.22.82.25
coredns-57dc86754b-9schh    1/1     Running       0          17d     172.22.82.25     cn-hangzhou.xxx.xxx.xxx.xxx    <none>[root@iZbp16er8wkobo2a165ekzZ ~]# kubectl logs -n kube-system coredns-57dc86754b-9schh | tail -10
2020-03-05T13:06:29.745Z [INFO] 172.22.0.66:54106 - 24311 "A IN metrics.cn-hangzhou.aliyuncs.com. udp 50 false 512" NOERROR qr,rd,ra 190 0.00003395s
2020-03-05T13:06:29.745Z [INFO] 172.22.0.66:56730 - 20278 "AAAA IN metrics.cn-hangzhou.aliyuncs.com. udp 50 false 512" NOERROR qr,rd,ra 232 0.000072002s
2020-03-05T13:06:29.748Z [INFO] 172.22.0.66:41047 - 53211 "AAAA IN metrics.cn-hangzhou.aliyuncs.com.kube-system.svc.cluster.local. udp 80 false 512" NXDOMAIN qr,rd,ra 173 0.00004196s
2020-03-05T13:06:29.748Z [INFO] 172.22.0.66:40581 - 40714 "A IN metrics.cn-hangzhou.aliyuncs.com.kube-system.svc.cluster.local. udp 80 false 512" NXDOMAIN qr,rd,ra 173 0.000037223s
2020-03-05T13:06:29.75Z [INFO] 172.22.0.66:41910 - 55424 "AAAA IN metrics.cn-hangzhou.aliyuncs.com.svc.cluster.local. udp 68 false 512" NXDOMAIN qr,rd,ra 161 0.000032467s
2020-03-05T13:06:29.75Z [INFO] 172.22.0.66:35103 - 22415 "A IN metrics.cn-hangzhou.aliyuncs.com.svc.cluster.local. udp 68 false 512" NXDOMAIN qr,rd,ra 161 0.000071336s
2020-03-05T13:06:29.752Z [INFO] 172.22.0.66:38912 - 45565 "AAAA IN metrics.cn-hangzhou.aliyuncs.com.cluster.local. udp 64 false 512" NXDOMAIN qr,rd,ra 157 0.000034122s
2020-03-05T13:06:29.752Z [INFO] 172.22.0.66:35033 - 24488 "A IN metrics.cn-hangzhou.aliyuncs.com.cluster.local. udp 64 false 512" NXDOMAIN qr,rd,ra 157 0.000040106s
2020-03-05T13:06:29.755Z [INFO] 172.22.0.66:50165 - 55872 "A IN metrics.cn-hangzhou.aliyuncs.com. udp 50 false 512" NOERROR qr,rd,ra 190 0.00004164s
2020-03-05T13:06:29.756Z [INFO] 172.22.0.66:37725 - 64492 "AAAA IN metrics.cn-hangzhou.aliyuncs.com. udp 50 false 512" NOERROR qr,rd,ra 232 0.000029312s
```
查看日志发现基本上都是172.22.0.66这个pod发起的metrics.cn-hangzhou.aliyuncs.com域名的解析，但是同一个pod解析同一个地址，有NXDOMAIN，又有NOERROR，于是尝试手动解析一下。分别去到coredns pod上做解析和172.22.0.66这个pod上手动解析，由于coredns pod镜像基本命令都不支持，无法通过命令登录：

```
[root@iZbp16er8wkobo2a165ekzZ ~]# kubectl exec -ti -n kube-system coredns-57dc86754b-9schh /bin/bash
OCI runtime exec failed: exec failed: container_linux.go:344: starting container process caused "exec: \"/bin/bash\": stat /bin/bash: no such file or directory": unknown command terminated with exit code 126
```

所以使用nsenter工具通过网络命名空间的方式进入：

```
# docker inspect 9cddee6aa78d |grep Pid
"Pid": 13464,
"PidMode": "",
"PidsLimit": 0,
# nsenter -t 13464 --net bash
# ifconfig   //这样看到的就是pod的IP了
```

手工解析发现每次都是解析正常的。通过和阿里云的同学沟通了解到alicloud-monitor-controller这个插件每次做解析都会有3条NXDOMAIN和1条NOERROR的记录，最后都是解析正常的。原因是解析k8s集群外部的域名是还会一次search集群内部的域。导致有3条解析失败的日志，最后没有匹配到集群内部的域而向外解析。可以通过统计集群外部域名和集群内部域名的解析日志验证：

```
[root@iZbp16er8wkobo2a165ekzZ ~]# kubectl logs -n kube-system coredns-57dc86754b-9schh | grep  cvte-ms-mc4.redis.rds.aliyuncs.com  | awk '{print $8}' | sort -rn |uniq -c
130 cvte-ms-mc4.redis.rds.aliyuncs.com.
[root@iZbp16er8wkobo2a165ekzZ ~]# kubectl logs -n kube-system coredns-57dc86754b-9schh | grep  "172.22.0.66" |grep metrics.cn-hangzhou.aliyuncs.com  | awk '{print $8}' | sort -rn |uniq -c
11572 metrics.cn-hangzhou.aliyuncs.com.svc.cluster.local.
11572 metrics.cn-hangzhou.aliyuncs.com.kube-system.svc.cluster.local.
11572 metrics.cn-hangzhou.aliyuncs.com.cluster.local.
11574 metrics.cn-hangzhou.aliyuncs.com.
```

可以看到，内部域名之后解析成功的记录，而外部域名会有3条k8s集群内部域解析失败的记录。从而可以验证，这说明alicloud_monitor-controller的dns解析方式不合理，会严重影响coredns的性能。应该优化。

查看172.22.0.66和coredns pod的/etc/resolv.conf文件分别如下：

```
//172.22.0.66的/etc/resolv.conf文件 [root@iZbp16er8wkobo2a165ekzZ ~]# kubectl exec -ti alicloud-monitor-controller-8bc847d8d-2t5c9 -n kube-system sh
/go $ cat /etc/resolv.conf
nameserver 172.23.0.10
search kube-system.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
/go $

//其他node节点pod的/etc/resolv.conf文件
app # cat /etc/resolv.conf
nameserver 172.23.0.10
search project-1602-pro-pro-new.svc.cluster.local svc.cluster.local cluster.local
options ndots:2
app #
```
发现文件中options选项的ndots参数配置不一样。ndots这个参数是做什么用的呢？

其中search kube-system.svc.cluster.local svc.cluster.local cluster.local和options ndots:5这两行配置表明，所有查询中，如果.的个数少于5个，则会根据search中配置的列表依次在对应域中先进行搜索，如果没有返回，则最后再直接查询域名本身。所以就会出现解析外部域名的时候会依次查询kube-system.svc.cluster.local，svc.cluster.local和cluster.local这三个域，多了3条NXDOMAIN的解析错误的记录。
至此，问题已经基本清晰。


### 解决

解决方法有以下三个：
1）通过修改172.22.0.6的/etc/resolv.conf文件的ndots参数为2，来实现。（注意这需要确认业务是否不需要解析集群内部的域名）
2）关闭alicloud-monitor-controller云监控插件。
3）尝试修改coredns的配置来实现，如下（需要验证）：

附：一则Corefile的configmap的配置：

```
# kubectl  get configmap -n kube-system coredns -o jsonpath='{.data}'

map[Corefile:.:53 {
    log
    errors
    health
    kubernetes cluster.local in-addr.arpa ip6.arpa {
       pods insecure
       upstream
       fallthrough in-addr.arpa ip6.arpa
    }
    hosts /etc/hostfile {
       fallthrough
    }
    proxy domain.com 10.111.xxx.1:53 10.111.xxx.2:53 {
       policy round_robin
    }
    proxy domain.cn 10.111.xxx.1:53 10.111.xxx.2:53 {
       policy round_robin
    }
    proxy consul 10.111.xxx.1:53 10.111.xxx.2:53 {
       policy round_robin
    }
    prometheus :9153
    proxy . /etc/resolv.conf {
       policy first
    }
    cache 120
    loop
    reload  //加上这个配置之后修改配置会自动reload pod的配置，可能存在pod不会重启
    loadbalance
}
]
```
参数使用介绍：

```
error: 错误记录到stdout
health ：CoreDNS的运行状况报告为 http：// localhost：8080 / health
kubernetes ：CoreDNS将根据Kubernetes服务和pod的IP回复DNS查询
prometheus ：CoreDNS的度量标准可以在 http://localhost:9153/ Prometheus格式的 指标 中找到
proxy ：任何不在Kubernetes集群域内的查询都将转发到预定义的解析器（/etc/resolv.conf）
cache ：启用前端缓存
loop ：检测简单的转发循环，如果找到循环则停止CoreDNS进程
reload ：允许自动重新加载已更改的Corefile。编辑ConfigMap配置后，请等待两分钟以使更改生效

loadbalance ：这是一个循环DNS负载均衡器，可以在答案中随机化A，AAAA和MX记录的顺序
```


使用命令，编辑修改，然后修改对应的deployment重启pod或者重新发布pod生效。
     
```
kubectl edit deployment -n kube-system coredns
```




