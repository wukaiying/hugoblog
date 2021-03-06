---
title: "K8S Api Server 源码分析"
date: 2020-04-13T20:31:49+08:00
categories: ["k8s"]
keywords: ["apiserver"]  
description: "k8s api server源码分析" 
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
### 版本环境
- kubernetes版本：kubernetes:v1.16.0

- go环境：go version go1.13.4 darwin/amd64

### Api Server架构

kubernetes api server 提供了增删改查，list,watch各类资源的接口，对请求进行验证，鉴权及admission controller, 请求成功后将结果更新到后端存储etcd（也可以是其他存储）中。

api server 架构从上到下可以分为以下几层：

![api server架构图](https://pic.images.ac.cn/image/5e9ea523190cc.html)

#### 1.Api层，主要包含三种api

core api: 主要在/api/v1下

分组api: 其path为/apis/$NAME/$VERSION

暴露系统状态的api: 如/metrics,/healthz,/logs,/openapi等

```bash
$ curl -k https://127.0.0.1:56521

{
  "paths": [
    "/api",
    "/api/v1",
    "/apis",
    "/apis/",
    "/apis/admissionregistration.k8s.io",
    "/apis/admissionregistration.k8s.io/v1beta1",
    ...
    "/healthz",
    "/healthz/autoregister-completion",
    "/healthz/etcd",
    ...
    "/logs",
    "/metrics",
    "/openapi/v2",
    "/swagger-2.0.0.json",
    "/swaggerapi",
    "/version"
```

#### 2.访问控制层

当用户访问api接口时，访问控制层将会做如下校验：

对用户进行身份校验(authentication)--> 校验用户对k8s资源对象访问权限(authorization) --> 根据配置的资源访问许可逻辑(admission control)，判断是否访问。

下面分别对这三种校验方式进行说明：

#### 2.1 Authentication 身份校验

k8s提供了三种客户端身份校验方式：https双向证书认证，http token方式认证， http base 认证也就是用户名密码方式

a. 基于CA根证书签名的双向数字证书认证

client和server向CA机构申请证书

client->server，server下发服务端证书，client利用证书验证server是否合法

client发送客户端证书给server，server利用证书校验client是否合法

client和server利用随机密钥加密消息，然后通信

apiserver启动的时候通过--client-ca-file=XXXX配置签发client证书的CA，当client发送证书过来时，apiserver使用CA验证，如果证书合法，则可进行通信。

这种方式的优点是安全，缺点是无法撤销用户证书，创建集群就绑定了证书。

```bash
$ k get pods kube-apiserver-kind-control-plane -n kube-system -o yaml

- command:
    - kube-apiserver
    - --authorization-mode=Node,RBAC
    - --advertise-address=172.17.0.2
    - --allow-privileged=true
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
```

b. http token 认证

生成一个长串token，并放入header发起http请求，apiserver依据token识别用户

apiserver启动通过--token-auth-file=XXXXX加载所有用户token，client请求的时候Header加上token即可

这种方式的有点是操作简单，缺点每次重新设置token需要重新启动apisever。
```bash
$ k get pods kube-apiserver-kind-control-plane -n kube-system -o yaml

- command:
   --token-auth-file string
```

c. http 用户名密码认证

在向api server发送请求时，将`username:password`进行base64加密存入http request中的Header Authorization域中，apiserver依据这个域中的信息识别用户。

apiserver启动时通过参数--basic-auth-file=XXXX指定用户信息的文件路径，client请求的时候Header带上base64(user:password)即可。

d. service account 

主要面向pod内部访问apiserver的鉴权方式。controller manager和apiserver会利用server端的私钥为每个pod创建一个token，并挂载到/run/secrets/kubernetes.io/serviceaccount/路径下。pod在访问apiserver的时候带上该token就行

```bash
$ k exec -it mysql2-65848bd47c-8p5jh /bin/bash

root@mysql2-65848bd47c-8p5jh:/run/secrets/kubernetes.io/serviceaccount# ls
ca.crt	namespace  token
```

e. 其他方式(TODO，暂时标记)

openId
基于OAuth2协议认证，受各大云提供商青睐

Webhook Token
按照规定k8s的接口规范，自定义token认证

keystone password
openstack提供的认证和授权方案，适合采用openstack体系搭建k8s的团队，处于试验阶段

匿名请求
当开启匿名请求时，所有没有被拒绝的请求都被认为是匿名请求，在授权阶段，这些请求都被统一处理

源码分析：

```bash
pkg/kubeapiserver/authenticator/config.go

# 根据传入认证方式信息来生成 authenticators，然后将用户信息提取进行校验
func (config Config) New() (authenticator.Request, *spec.SecurityDefinitions, error) {
	var authenticators []authenticator.Request
    // basic auth
	if len(config.BasicAuthFile) > 0 {
		basicAuth, err := newAuthenticatorFromBasicAuthFile(config.BasicAuthFile)
		if err != nil {
			return nil, nil, err
		}
		authenticators = append(authenticators, authenticator.WrapAudienceAgnosticRequest(config.APIAudiences, basicAuth))

		securityDefinitions["HTTPBasic"] = &spec.SecurityScheme{
			SecuritySchemeProps: spec.SecuritySchemeProps{
				Type:        "basic",
				Description: "HTTP Basic authentication",
			},
		}
	}

	// X509 methods
	if len(config.ClientCAFile) > 0 {
		certAuth, err := newAuthenticatorFromClientCAFile(config.ClientCAFile)
		if err != nil {
			return nil, nil, err
		}
		authenticators = append(authenticators, certAuth)
	}

	// Bearer token methods, local first, then remote
	if len(config.TokenAuthFile) > 0 {
		tokenAuth, err := newAuthenticatorFromTokenFile(config.TokenAuthFile)
		if err != nil {
			return nil, nil, err
		}
		tokenAuthenticators = append(tokenAuthenticators, authenticator.WrapAudienceAgnosticToken(config.APIAudiences, tokenAuth))
	}
	if len(config.ServiceAccountKeyFiles) > 0 {
		serviceAccountAuth, err := newLegacyServiceAccountAuthenticator(config.ServiceAccountKeyFiles, config.ServiceAccountLookup, config.APIAudiences, config.ServiceAccountTokenGetter)
		if err != nil {
			return nil, nil, err
		}
		tokenAuthenticators = append(tokenAuthenticators, serviceAccountAuth)
	}

    fmt.Println("the length of authticator:", len(authenticators))
    switch len(authenticators) {
    case 0:
        return nil, nil
    case 1:
        return authenticators[0], nil
    default:
        return union.New(authenticators...), nil
    }

}
```
#### 2.2 Authorization 资源对象访问权限

通过api server用户认证的请求，接下来会进行资源访问权限认证，授予不同用户不同的资源访问权限。
通过--authorization-mode来设置，可以看到默认为Node，RBAC模式

```bash
$ k get pods kube-apiserver-kind-control-plane -n kube-system -o yaml

  containers:
  - command:
    - kube-apiserver
    - --authorization-mode=Node,RBAC
```

目前支持的几种授权策略：
- AlwaysDeny：拒绝所有请求，用于测试
- AlwaysAllow：允许所有请求，k8s默认策略
- ABAC：基于属性的访问控制。表示使用用户配置的规则对用户请求进行控制
- RBAC：基于角色的访问控制
- Webhook：通过调用外部rest服务对用户授权

*ABAC 介绍*
ABAC 配置需要两步第一步在api server启动参数中设置`--authorization-mode=ABAC`,然后指定策略文件的
路径和名称`--authorization-policy-file=SOME_FILENAME                       `(json格式)并重新启动api server。

策略文件中定义了某用户能放资源的权限，当用户再向api server提交资源创建属性后，api server会根据策略文件
中定义的策略，对用户提交的属性逐一进行判定(可能有多个策略)，如果至少有一条策略匹配成功，则该请求会被通过。

允许用户alice对所有资源做任何操作：
```json
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"user": "alice", "namespace": "*", "resource": "*", "apiGroup": "*"}}
```
允许kubelet读取任何pod
```json
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"user": "kubelet", "namespace": "*", "resource": "pods", "readonly": true}}
```
Bob 可以读取命名空间projectCaribou中的pod:
```json
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"user": "bob", "namespace": "projectCaribou", "resource": "pods", "readonly": true}}
```
一个完成的abac配置文件如下：
```json
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"group":"system:authenticated",  "nonResourcePath": "*", "readonly": true}}
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"group":"system:unauthenticated", "nonResourcePath": "*", "readonly": true}}
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"user":"admin",     "namespace": "*",              "resource": "*",         "apiGroup": "*"                   }}
...
```
*RBAC 介绍*
 
基于角色的权限控制系统，k8s推荐使用该模式，默认api server也是使用该模式

RBAC的四个资源对象：
- Role：一个角色就是一组权限的集合，拥有某个namespace下的权限
- ClusterRole：集群角色，拥有整个cluster下的权限
- RoleBinding：将Role绑定目标（user、group、service account）
- ClusterRoleBinding：将ClusterRoleBinding绑定目标

*webhook 授权介绍*

启用该模式进行授权认证，k8s会调用外部rest服务来进行授权。webhook 配置需要两步第一步在api server启动参数中设置`--authorization-mode=Webhook`，然后使用参数`--authorization-webhook-config-file=SOME_FILENAME`来设置远端授权服务的信息。
启用该模式后api server 相当于客户端，而外部rest server相当于服务端。

webhook配置文件样例：

```bash
# Kubernetes API version
apiVersion: v1
# kind of the API object
kind: Config
# 远端webhook service访问地址及证书.
clusters:
  - name: name-of-remote-authz-service
    cluster:
      # CA for verifying the remote service.
      certificate-authority: /path/to/ca.pem
      # URL of remote service to query. Must use 'https'. May not include parameters.
      server: https://authz.example.com/authorize

# users refers to the API Server's webhook configuration.
users:
  - name: name-of-api-server
    user:
      client-certificate: /path/to/cert.pem # cert for the webhook plugin to use
      client-key: /path/to/key.pem          # key matching the cert

# kubeconfig files require a context. Provide one for the API Server.
current-context: webhook
contexts:
- context:
    cluster: name-of-remote-authz-service
    user: name-of-api-server
  name: webhook
```

假设bob想要访问某个namespace下的pod, 在授权开始时，API Server会生成一个api.authorization.v1beta1.SubjectAccessReview对象，用于描述操作信息
```json
{
  "apiVersion": "authorization.k8s.io/v1beta1",
  "kind": "SubjectAccessReview",
  "spec": {
    "resourceAttributes": {
      "namespace": "kittensandponies",
      "verb": "get",
      "group": "unicorn.example.org",
      "resource": "pods"
    },
    "user": "jane",
    "group": [
      "group1",
      "group2"
    ]
  }
}
```
api server 会携带该信息向远端rest server发送post请求，远端rest server通过返回allow或者not allow来决定这次请求是否合法。
```json
{
  "apiVersion": "authorization.k8s.io/v1beta1",
  "kind": "SubjectAccessReview",
  "status": {
    "allowed": true/false
  }
}
```

#### 2.3 admission control 准入控制

经过认证和授权后，还需要经过多个准入控制器的审核，client到apiserver的请求才能成功，目前提供的准入控制器有这些（这些插件并不是都启用了）：
admission control 有一个插件列表，有k8s默认提供的，也有用户可以扩展的。
```
NamespaceLifecycle, LimitRanger, ServiceAccount, TaintNodesByCondition, Priority, DefaultTolerationSeconds, DefaultStorageClass, StorageObjectInUseProtection, PersistentVolumeClaimResize, MutatingAdmissionWebhook, ValidatingAdmissionWebhook, RuntimeClass, ResourceQuota
```
查看k8s支持的admission control插件进入api server，运行命令
```
kube-apiserver -h | grep enable-admission-plugins
```
比如上面有我常用到的两个`MutatingAdmissionWebhook, ValidatingAdmissionWebhook`来实现对用户提交的数据进行校验和patch。


#### 3.注册表层
Kubernetes把所有资源对象都保存在注册表（Registry）中，针对注册表中的各种资源对象都定义了：资源对象的类型、如何创建资源对象、如何转换资源的 不同版本，以及如何将资源编码和解码为JSON或ProtoBuf格式进行存储。

#### 4.etcd数据库
用于持久化存储Kubernetes资源对象的KV数据库，只有api server可以向etcd中写入数据。


### pod 启动完整流程

![pod创建流程](https://pic.images.ac.cn/image/5e9ea5288d013.html)

kubectl创建replicaset提交给api server，api server经过用户验证，权限验证，addmission验证，都通过之后提交到etcd。
etcd有资源创建后，会发送create事件给api server，api server自己也有list-watch接口，这样其他组件可以通过该接口监听到api server中的变化。
接下来,controller mannager 监听到有新的资源创建，开始创建pod，然后api sever将创建的pod存入etcd，etcd 发送创建pod事件给api server,
schedule 监听api server，发现有新的pod创建需要进行调度，通过预选，优选算法给pod选择合适的node，然后更新pod，api server将更新的pod写入etcd，
node节点中的kubelet监听api sever，之关注调度给自己的pod，发现有新的pod需要创建，开始在node中启动pod，然后将pod状态发送给api server， api server将pod更新后的信息写入etcd。

如何理解list-watch
list-watch用实现数据同步，为了减少api server压力，客户端首先调用API Server的List接口获取相关资源对象的全量数据并将其缓存到内存中，然后启动对应资源 对象的Watch协程，
在接收到Watch事件后，再根据事件的类型（比如新增、修改或删 除）对内存中的全量资源对象列表做出相应的同步修改。


### 参考
https://mp.weixin.qq.com/s/TQuqAAzBjeWHwKPJZ3iJhA

https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/

https://my.oschina.net/u/1378920/blog/1807839

https://www.jianshu.com/p/daa4ff387a78