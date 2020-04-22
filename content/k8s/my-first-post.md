---
title: "Kubernetes中leaderelection实现组件高可用"
date: 2020-02-11T10:01:25+08:00
tags: ["k8s", "elect"]  
categories: ["k8s"]
keywords: ["leaderelection"]  
description: "Kubernetes中leaderelection实现组件高可用" 
summaryLength: 100
draft: false
---

在Kubernetes中，通常kube-schduler和kube-controller-manager都是多副本进行部署的来保证高可用，而真正在工作的实例其实只有一个。

这里就利用到 leaderelection 的选主机制，保证leader是处于工作状态，并且在leader挂掉之后，从其他节点选取新的leader保证组件正常工作。今天就来看看这个包的使用以及它内部是如何实现的。

### 基本使用

以下是一个简单使用的例子，编译完成之后同时启动多个进程，但是只有一个进程在工作，当把leader进程kill掉之后，会重新选举出一个leader进行工作，即执行其中的 run 方法：

```go
/*
例子来源于client-go中的example包中，
*/
package main
import (
	"context"
	"flag"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/google/uuid"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	clientset "k8s.io/client-go/kubernetes"
	"k8s.io/client-go/rest"
	"k8s.io/client-go/tools/clientcmd"
	"k8s.io/client-go/tools/leaderelection"
	"k8s.io/client-go/tools/leaderelection/resourcelock"
	"k8s.io/klog"
)

func buildConfig(kubeconfig string) (*rest.Config, error) {
	if kubeconfig != "" {
		cfg, err := clientcmd.BuildConfigFromFlags("", kubeconfig)
		if err != nil {
			returnnil, err
		}
		return cfg, nil
	}

	cfg, err := rest.InClusterConfig()
	if err != nil {
		returnnil, err
	}
	return cfg, nil
}

func main() {
	klog.InitFlags(nil)

	var kubeconfig string
	var leaseLockName string
	var leaseLockNamespace string
	var id string

	flag.StringVar(&kubeconfig, "kubeconfig", "", "absolute path to the kubeconfig file")
	flag.StringVar(&id, "id", uuid.New().String(), "the holder identity name")
	flag.StringVar(&leaseLockName, "lease-lock-name", "", "the lease lock resource name")
	flag.StringVar(&leaseLockNamespace, "lease-lock-namespace", "", "the lease lock resource namespace")
	flag.Parse()

	if leaseLockName == "" {
		klog.Fatal("unable to get lease lock resource name (missing lease-lock-name flag).")
	}
	if leaseLockNamespace == "" {
		klog.Fatal("unable to get lease lock resource namespace (missing lease-lock-namespace flag).")
	}

	// leader election uses the Kubernetes API by writing to a
	// lock object, which can be a LeaseLock object (preferred),
	// a ConfigMap, or an Endpoints (deprecated) object.
	// Conflicting writes are detected and each client handles those actions
	// independently.
	config, err := buildConfig(kubeconfig)
	if err != nil {
		klog.Fatal(err)
	}
	client := clientset.NewForConfigOrDie(config)

	run := func(ctx context.Context) {
		// complete your controller loop here
		klog.Info("Controller loop...")

		select {}
	}

	// use a Go context so we can tell the leaderelection code when we
	// want to step down
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	// listen for interrupts or the Linux SIGTERM signal and cancel
	// our context, which the leader election code will observe and
	// step down
	ch := make(chan os.Signal, 1)
	signal.Notify(ch, os.Interrupt, syscall.SIGTERM)
	gofunc() {
		<-ch
		klog.Info("Received termination, signaling shutdown")
		cancel()
	}()

	// we use the Lease lock type since edits to Leases are less common
	// and fewer objects in the cluster watch "all Leases".
    // 指定锁的资源对象，这里使用了Lease资源，还支持configmap，endpoint，或者multilock(即多种配合使用)
	lock := &resourcelock.LeaseLock{
		LeaseMeta: metav1.ObjectMeta{
			Name:      leaseLockName,
			Namespace: leaseLockNamespace,
		},
		Client: client.CoordinationV1(),
		LockConfig: resourcelock.ResourceLockConfig{
			Identity: id,
		},
	}

	// start the leader election code loop
	leaderelection.RunOrDie(ctx, leaderelection.LeaderElectionConfig{
		Lock: lock,
		// IMPORTANT: you MUST ensure that any code you have that
		// is protected by the lease must terminate **before**
		// you call cancel. Otherwise, you could have a background
		// loop still running and another process could
		// get elected before your background loop finished, violating
		// the stated goal of the lease.
		ReleaseOnCancel: true,
		LeaseDuration:   60 * time.Second,//租约时间
		RenewDeadline:   15 * time.Second,//更新租约的
		RetryPeriod:     5 * time.Second,//非leader节点重试时间
		Callbacks: leaderelection.LeaderCallbacks{
			OnStartedLeading: func(ctx context.Context) {
                //变为leader执行的业务代码
				// we're notified when we start - this is where you would
				// usually put your code
				run(ctx)
			},
			OnStoppedLeading: func() {
                 // 进程退出
				// we can do cleanup here
				klog.Infof("leader lost: %s", id)
				os.Exit(0)
			},
			OnNewLeader: func(identity string) {
                //当产生新的leader后执行的方法
				// we're notified when new leader elected
				if identity == id {
					// I just got the lock
					return
				}
				klog.Infof("new leader elected: %s", identity)
			},
		},
	})
}
```
#### 关键启动参数

```
kubeconfig: 指定kubeconfig文件地址
lease-lock-name：指定lock的名称
lease-lock-namespace：指定lock存储的namespace
id: 例子中提供的区别参数，用于区分实例
logtostderr：klog提供的参数，指定log输出到控制台
v: 指定日志输出级别
```

#### 启动多个进程

进程1：
```
go run main.go -kubeconfig=/Users/silenceper/.kube/config -logtostderr=true -lease-lock-name=example -lease-lock-namespace=default -id=1 -v=4
I0215 19:56:37.049658   48045 leaderelection.go:242] attempting to acquire leader lease  default/example...
I0215 19:56:37.080368   48045 leaderelection.go:252] successfully acquired lease default/example
I0215 19:56:37.080437   48045 main.go:87] Controller loop...
```

进程2：
```
go run main.go -kubeconfig=/Users/silenceper/.kube/config -logtostderr=true -lease-lock-name=example -lease-lock-namespace=default -id=2 -v=4
I0215 19:56:37.049658   48045 leaderelection.go:242] attempting to acquire leader lease  default/example...
I0215 19:56:37.080368   48045 leaderelection.go:252] successfully acquired lease default/example
I0215 19:56:37.080437   48045 main.go:87] Controller loop...
```
这里可以看出来id=1的进程持有锁，并且运行的程序，而id=2的进程表示无法获取到锁，在不断的进程尝试。

现在kill掉id=1进程，在等待lock释放之后（有个LeaseDuration时间），leader变为id=2的进程执行工作。

```
I0215 20:01:41.489300   48791 leaderelection.go:252] successfully acquired lease default/example
I0215 20:01:41.489577   48791 main.go:87] Controller loop...
```

### 源码分析
基本原理其实就是利用通过Kubernetes中configmap，endpoints或者lease资源实现一个分布式锁，抢(acqure)到锁的节点成为leader，并且定期更新（renew）。其他进程也在不断的尝试进行抢占，抢占不到则继续等待下次循环。当leader节点挂掉之后，租约到期，其他节点就成为新的leader。


#### 入口
通过 leaderelection.RunOrDie 启动

```go
func RunOrDie(ctx context.Context, lec LeaderElectionConfig) {
	le, err := NewLeaderElector(lec)
	if err != nil {
		panic(err)
	}
	if lec.WatchDog != nil {
		lec.WatchDog.SetLeaderElection(le)
	}
	le.Run(ctx)
}
```
传入参数 LeaderElectionConfig:

```go
type LeaderElectionConfig struct {
	// Lock 的类型
	Lock rl.Interface
	//持有锁的时间
	LeaseDuration time.Duration
	//在更新租约的超时时间
	RenewDeadline time.Duration
    //竞争获取锁的时间
	RetryPeriod time.Duration
    //状态变化时执行的函数，支持三种：
    //1、OnStartedLeading 启动是执行的业务代码
    //2、OnStoppedLeading leader停止执行的方法
    //3、OnNewLeader 当产生新的leader后执行的方法
	Callbacks LeaderCallbacks

    //进行监控检查
	// WatchDog is the associated health checker
	// WatchDog may be null if its not needed/configured.
	WatchDog *HealthzAdaptor
    //leader退出时，是否执行release方法
	ReleaseOnCancel bool
    
	// Name is the name of the resource lock for debugging
	Name string
}
```
LeaderElectionConfig.lock 支持保存在以下三种资源中：
```
configmap 
endpoint 
lease 
```
包中还提供了一个multilock即可以进行选择两种，当其中一种保存失败时，选择第二中，可以在interface.go中看到：
```go
switch lockType {
	case EndpointsResourceLock://保存在endpoints
		return endpointsLock, nil
	case ConfigMapsResourceLock://保存在configmaps
		return configmapLock, nil
	case LeasesResourceLock://保存在leases
		return leaseLock, nil
	case EndpointsLeasesResourceLock://优先尝试保存在endpoint失败时保存在lease
		return &MultiLock{
			Primary:   endpointsLock,
			Secondary: leaseLock,
		}, nil
	case ConfigMapsLeasesResourceLock://优先尝试保存在configmap，失败时保存在lease
		return &MultiLock{
			Primary:   configmapLock,
			Secondary: leaseLock,
		}, nil
	default:
		returnnil, fmt.Errorf("Invalid lock-type %s", lockType)
	}
```

以lease资源对象为例，可以在查看到保存的内容:
```
$ kubectl get lease example -n default -o yaml
apiVersion:coordination.k8s.io/v1
kind:Lease
metadata:
  creationTimestamp:"2020-02-15T11:56:37Z"
  name:example
  namespace:default
  resourceVersion:"210675"
  selfLink:/apis/coordination.k8s.io/v1/namespaces/default/leases/example
  uid:a3470a06-6fc3-42dc-8242-9d6cebdf5315
spec:
  acquireTime:"2020-02-15T12:01:41.476971Z"//获得锁时间
  holderIdentity:"2"//持有锁进程的标识
  leaseDurationSeconds:60//lease租约
  leaseTransitions:1//leader更换次数
  renewTime:"2020-02-15T12:05:37.134655Z"//更新租约的时间
```
关注其spec中的字段，分别进行标注,对应结构体如下：

```go
type LeaderElectionRecord struct {
	HolderIdentity       string`json:"holderIdentity"`//持有锁进程的标识，一般可以利用主机名
	LeaseDurationSeconds int`json:"leaseDurationSeconds"`//  lock的租约
	AcquireTime          metav1.Time `json:"acquireTime"`//持有锁的时间
	RenewTime            metav1.Time `json:"renewTime"`//更新时间
	LeaderTransitions    int`json:"leaderTransitions"`//leader更换的次数
}
```
#### 获取的锁以及更新锁

Run方法中包含了获取锁以及更新锁的入口

```go
// Run starts the leader election loop
func (le *LeaderElector) Run(ctx context.Context) {
	deferfunc() {
        //进行退出执行
		runtime.HandleCrash()
        //停止时执行回调方法
		le.config.Callbacks.OnStoppedLeading()
	}()
    //不断的进行获得锁，如果获得锁成功则执行后面的方法，否则不断的进行重试
	if !le.acquire(ctx) {
		return// ctx signalled done
	}
	ctx, cancel := context.WithCancel(ctx)
	defer cancel()
    //获取锁成功，当前进程变为leader，执行回调函数中的业务代码
	go le.config.Callbacks.OnStartedLeading(ctx)
    //不断的循环进行进行租约的更新，保证锁一直被当前进行持有
	le.renew(ctx)
}
```
le.acquire 和 le.renew 内部都是调用了 le.tryAcquireOrRenew 函数，只是对于返回结果的处理不一样。

le.acquire 对于 le.tryAcquireOrRenew 返回成功则退出，失败则继续。

le.renew 则相反，成功则继续，失败则退出。

我们来看看 tryAcquireOrRenew 方法：

```go
func (le *LeaderElector) tryAcquireOrRenew() bool {
	now := metav1.Now()
    //锁资源对象内容
	leaderElectionRecord := rl.LeaderElectionRecord{
		HolderIdentity:       le.config.Lock.Identity(),//唯一标识
		LeaseDurationSeconds: int(le.config.LeaseDuration / time.Second),
		RenewTime:            now,
		AcquireTime:          now,
	}

	// 1. obtain or create the ElectionRecord
    // 第一步：从k8s资源中获取原有的锁
	oldLeaderElectionRecord, oldLeaderElectionRawRecord, err := le.config.Lock.Get()
	if err != nil {
		if !errors.IsNotFound(err) {
			klog.Errorf("error retrieving resource lock %v: %v", le.config.Lock.Describe(), err)
			returnfalse
		}
        //资源对象不存在，进行锁资源创建
		if err = le.config.Lock.Create(leaderElectionRecord); err != nil {
			klog.Errorf("error initially creating leader election record: %v", err)
			returnfalse
		}
		le.observedRecord = leaderElectionRecord
		le.observedTime = le.clock.Now()
		returntrue
	}

	// 2. Record obtained, check the Identity & Time
    // 第二步，对比存储在k8s中的锁资源与上一次获取的锁资源是否一致
	if !bytes.Equal(le.observedRawRecord, oldLeaderElectionRawRecord) {
		le.observedRecord = *oldLeaderElectionRecord
		le.observedRawRecord = oldLeaderElectionRawRecord
		le.observedTime = le.clock.Now()
	}
    //判断持有的锁是否到期以及是否被自己持有
	iflen(oldLeaderElectionRecord.HolderIdentity) > 0 &&
		le.observedTime.Add(le.config.LeaseDuration).After(now.Time) &&
		!le.IsLeader() {
		klog.V(4).Infof("lock is held by %v and has not yet expired", oldLeaderElectionRecord.HolderIdentity)
		returnfalse
	}

	// 3. We're going to try to update. The leaderElectionRecord is set to it's default
	// here. Let's correct it before updating.
    //第三步：自己现在是leader，但是分两组情况，上一次也是leader和首次变为leader
	if le.IsLeader() {
        //自己本身就是leader则不需要更新AcquireTime和LeaderTransitions
		leaderElectionRecord.AcquireTime = oldLeaderElectionRecord.AcquireTime
		leaderElectionRecord.LeaderTransitions = oldLeaderElectionRecord.LeaderTransitions
	} else {
        //首次自己变为leader则更新leader的更换次数
		leaderElectionRecord.LeaderTransitions = oldLeaderElectionRecord.LeaderTransitions + 1
	}

    //更新锁资源，这里如果在 Get 和 Update 之间有变化，将会更新失败
	// update the lock itself
	if err = le.config.Lock.Update(leaderElectionRecord); err != nil {
		klog.Errorf("Failed to update lock: %v", err)
		returnfalse
	}

	le.observedRecord = leaderElectionRecord
	le.observedTime = le.clock.Now()
	returntrue
}
```
在这一步如果发生并发操作怎么样？

这里很重要一点就是利用到了k8s api操作的原子性：

在 le.config.Lock.Get() 中会获取到锁的对象，其中有一个 resourceVersion 字段用于标识一个资源对象的内部版本，每次更新操作都会更新其值。如果一个更新操作附加上了 resourceVersion 字段，那么 apiserver 就会通过验证当前 resourceVersion 的值与指定的值是否相匹配来确保在此次更新操作周期内没有其他的更新操作，从而保证了更新操作的原子性。

### 总结

leaderelection 主要是利用了k8s API操作的原子性实现了一个分布式锁，在不断的竞争中进行选举。

选中为leader的进行才会执行具体的业务代码，这在k8s中非常的常见，而且我们很方便的利用这个包完成组件的编写，从而实现组件的高可用，比如部署为一个多副本的Deployment，当leader的pod退出后会重新启动，可能锁就被其他pod获取继续执行。





