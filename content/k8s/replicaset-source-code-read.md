---
title: "Replica Set Controller 源码分析"
date: 2020-04-21T15:53:20+08:00
draft: false
categories: ["k8s"]
keywords: ["replicaset"]  
description: "replicaset controller源码分析" 
summaryLength: 100
---

### 版本环境
kubernetes版本：kubernetes:v1.16.0

go环境：go version go1.13.4 darwin/amd64

### 前言

前面文章已经总结到，deployment controller通过replicaset来控制pod的创建和删除，进而实现更高级操作例如，滚动更新，回滚等。本文继续从源码角度
分析replicaset controller如何控制pod的创建和删除。

在分析源码前先考虑一下 replicaset 的使用场景，在平时的操作中其实我们并不会直接操作 replicaset，replicaset 也仅有几个简单的操作，创建、删除、更新等。


### 源码分析

#### 入口

我们在上一篇文章中也说道controller manager是所有k8s 资源controller的管控中心，包含的主要逻辑就是高可用选择leader，然后
遍历所有的controllers,对每一个controller进行初始化并启动，也就是说只要集群启动，所有的controller也会都启动起来，开始list-watch api server，来准备创建资源。

```go
cmd/kube-controller-manager/app/controllermanager.go:386
func NewControllerInitializers(loopMode ControllerLoopMode) map[string]InitFunc {
	controllers := map[string]InitFunc{}
	controllers["endpoint"] = startEndpointController
	controllers["endpointslice"] = startEndpointSliceController
	...
	controllers["deployment"] = startDeploymentController
	controllers["replicaset"] = startReplicaSetController
}
```
接下来进入`startReplicaSetController`, 发现通过`NewReplicaSetController`初始化了ReplicaSetController，并并调用run方法启动controller。

```go
cmd/kube-controller-manager/app/apps.go:69

func startReplicaSetController(ctx ControllerContext) (http.Handler, bool, error) {
	if !ctx.AvailableResources[schema.GroupVersionResource{Group: "apps", Version: "v1", Resource: "replicasets"}] {
		return nil, false, nil
	}
	go replicaset.NewReplicaSetController(
		ctx.InformerFactory.Apps().V1().ReplicaSets(),
		ctx.InformerFactory.Core().V1().Pods(),
		ctx.ClientBuilder.ClientOrDie("replicaset-controller"),
		replicaset.BurstReplicas,
	).Run(int(ctx.ComponentConfig.ReplicaSetController.ConcurrentRSSyncs), ctx.Stop)
	return nil, true, nil
}
```

我们先分析`NewReplicaSetController`看一下初始化`ReplicaSetController`的具体步骤：发现他会监听rs的add,update,delete操作，和pod的add,update,delete操作。
当这两种资源发送变化时，将key(name+namespace)添加到queue队列中去。

```go
//pkg/controller/replicaset/replica_set.go:126

func NewBaseController(rsInformer appsinformers.ReplicaSetInformer, podInformer coreinformers.PodInformer, kubeClient clientset.Interface, burstReplicas int,
	gvk schema.GroupVersionKind, metricOwnerName, queueName string, podControl controller.PodControlInterface) *ReplicaSetController {
	if kubeClient != nil && kubeClient.CoreV1().RESTClient().GetRateLimiter() != nil {
		metrics.RegisterMetricAndTrackRateLimiterUsage(metricOwnerName, kubeClient.CoreV1().RESTClient().GetRateLimiter())
	}

	rsc := &ReplicaSetController{
		GroupVersionKind: gvk,
		kubeClient:       kubeClient,
		podControl:       podControl,
		burstReplicas:    burstReplicas,
		expectations:     controller.NewUIDTrackingControllerExpectations(controller.NewControllerExpectations()),
		queue:            workqueue.NewNamedRateLimitingQueue(workqueue.DefaultControllerRateLimiter(), queueName),
	}

	rsInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc:    rsc.enqueueReplicaSet,
		UpdateFunc: rsc.updateRS,
		
		DeleteFunc: rsc.enqueueReplicaSet,
	})
	rsc.rsLister = rsInformer.Lister()
	rsc.rsListerSynced = rsInformer.Informer().HasSynced

	podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc: rsc.addPod,
		
		UpdateFunc: rsc.updatePod,
		DeleteFunc: rsc.deletePod,
	})
	rsc.podLister = podInformer.Lister()
	rsc.podListerSynced = podInformer.Informer().HasSynced

	rsc.syncHandler = rsc.syncReplicaSet

	return rsc
}
```

接着我们再看`NewReplicaSetController`的Run方法：可以看到run方法会启动worker(默认5个)数量的goroutine来从queue中取出key,然后执行synchanlder操作。
下面会详细说明synchandler中的具体逻辑。补充一句，大部分contoller逻辑都是如此，流程就是注册name+startcontrollerhandler到 controller manager中，比如`controllers["replicaset"] = startReplicaSetController`
controller manager 会循环启动所有已经注册的name+startcontrollerhandler，对于每一个controller中的具体逻辑，又是通过NewxxxController.Run()的方式执行，
NewxxxCtronller是为了初始化controller，指定controller中需要监听的资源(比如，replicasetcontroller 他需要监听replicaset，pod这个两个资源的变化，而deploymentcontroller 它需要监听deployment,replicaset,pod这三个资源的变化, 具体变化包括add,update,delete 
这三种操作，每个操作可以写相应的逻辑来更新informer catche，修改完成后将key入队)
初始化完资源后，调用Run方法，启动controller，此时Controller会启动指定数量个worker goroutine来从informer队列中拿到key，获取informer catche中的资源和实际集群中的资源进行比较，
此时就进入了每个worker goroutine的synchandler逻辑当中。synchanlder就是核心谐调逻辑。

```go
//pkg/controller/replicaset/replica_set.go:177

func (rsc *ReplicaSetController) Run(workers int, stopCh <-chan struct{}) {
	defer utilruntime.HandleCrash()
	defer rsc.queue.ShutDown()

	controllerName := strings.ToLower(rsc.Kind)
	klog.Infof("Starting %v controller", controllerName)
	defer klog.Infof("Shutting down %v controller", controllerName)

	if !cache.WaitForNamedCacheSync(rsc.Kind, stopCh, rsc.podListerSynced, rsc.rsListerSynced) {
		return
	}

	for i := 0; i < workers; i++ {
		go wait.Until(rsc.worker, time.Second, stopCh)
	}

	<-stopCh
}

//pkg/controller/replicaset/replica_set.go:432
//每一个worker goroutine都会从informer队列中取出key,然后执行synchandler操作。

func (rsc *ReplicaSetController) worker() {
	for rsc.processNextWorkItem() {
	}
}

func (rsc *ReplicaSetController) processNextWorkItem() bool {
	key, quit := rsc.queue.Get()
	if quit {
		return false
	}
	defer rsc.queue.Done(key)

	err := rsc.syncHandler(key.(string))
	if err == nil {
		rsc.queue.Forget(key)
		return true
	}

	utilruntime.HandleError(fmt.Errorf("Sync %q failed with %v", key, err))
	rsc.queue.AddRateLimited(key)

	return true
}
```

synchandler中描述了特定类型controller的核心操作逻辑，不断的从informer 队列中拿到key, 对informer catche和当前集群实际资源，并进行比较，以达到期望。

```go
//pkg/controller/replicaset/replica_set.go:552

func (rsc *ReplicaSetController) syncReplicaSet(key string) error {

	startTime := time.Now()
	defer func() {
		klog.V(4).Infof("Finished syncing %v %q (%v)", rsc.Kind, key, time.Since(startTime))
	}()

    //1.通过key获取namespace, name
	namespace, name, err := cache.SplitMetaNamespaceKey(key)
	if err != nil {
		return err
	}
    
    //获取catche中的rs
	rs, err := rsc.rsLister.ReplicaSets(namespace).Get(name)
	if errors.IsNotFound(err) {
		klog.V(4).Infof("%v %v has been deleted", rsc.Kind, key)
		rsc.expectations.DeleteExpectations(key)
		return nil
	}
	if err != nil {
		return err
	}

	rsNeedsSync := rsc.expectations.SatisfiedExpectations(key)
	selector, err := metav1.LabelSelectorAsSelector(rs.Spec.Selector)
	if err != nil {
		utilruntime.HandleError(fmt.Errorf("Error converting pod selector to selector: %v", err))
		return nil
	}

	// list all pods to include the pods that don't match the rs`s selector
	// anymore but has the stale controller ref.
	// TODO: Do the List and Filter in a single pass, or use an index.
	allPods, err := rsc.podLister.Pods(rs.Namespace).List(labels.Everything())
	if err != nil {
		return err
	}
	// Ignore inactive pods.
	filteredPods := controller.FilterActivePods(allPods)

	// NOTE: filteredPods are pointing to objects from cache - if you need to
	// modify them, you need to copy it first.

    //筛选出和当前rs有关联的所有pod
	filteredPods, err = rsc.claimPods(rs, selector, filteredPods)
	if err != nil {
		return err
	}

    //对比当前集群pod数量和informer cache中replicaset.spec.repica的大小，进行创建或者删除pod
	var manageReplicasErr error
	if rsNeedsSync && rs.DeletionTimestamp == nil {
		manageReplicasErr = rsc.manageReplicas(filteredPods, rs)
	}
	rs = rs.DeepCopy()
	newStatus := calculateStatus(rs, filteredPods, manageReplicasErr)

	// Always updates status as pods come up or die.
    //最后更新replicaset 状态
	updatedRS, err := updateReplicaSetStatus(rsc.kubeClient.AppsV1().ReplicaSets(rs.Namespace), rs, newStatus)
	if err != nil {
		// Multiple things could lead to this update failing. Requeuing the replica set ensures
		// Returning an error causes a requeue without forcing a hotloop
		return err
	}
	// Resync the ReplicaSet after MinReadySeconds as a last line of defense to guard against clock-skew.
	if manageReplicasErr == nil && updatedRS.Spec.MinReadySeconds > 0 &&
		updatedRS.Status.ReadyReplicas == *(updatedRS.Spec.Replicas) &&
		updatedRS.Status.AvailableReplicas != *(updatedRS.Spec.Replicas) {
		rsc.enqueueReplicaSetAfter(updatedRS, time.Duration(updatedRS.Spec.MinReadySeconds)*time.Second)
	}
	return manageReplicasErr
}
```

看一下syncReplicaset中最核心逻辑`manageReplicas`，主要步骤如下：
1.  计算出实际pod数量和期望pod数量的差值
2. 如果差值小于0，需要创建新的pod，通过slowStartBatch批量创建pod，返回成功创建的数量
2.1  //通过slowStartBatch批量创建pod，返回成功创建的数量
2.2 //对比差异，看还有缺多个pod没有创建
2.3 如果真的还有pod没有创建，则会在下一次谐调循环中继续从1开始
3. //如果diff 大于0，则说明需要进行删除pod
3.1 // 通过一定的策略返回需要删除的pod,	 not-ready < ready, unscheduled < scheduled, and pending < running. 
3.2 //执行删除操作
3.3 //如果某个一个goroutine有错误，则返回错误，会进入下次一次谐调循环
4. 下次循环进入，直到达到期望

```go
//pkg/controller/replicaset/replica_set.go:459

func (rsc *ReplicaSetController) manageReplicas(filteredPods []*v1.Pod, rs *apps.ReplicaSet) error {
    //计算出实际pod数量和期望pod数量的差值
	diff := len(filteredPods) - int(*(rs.Spec.Replicas))
	rsKey, err := controller.KeyFunc(rs)
	if err != nil {
		utilruntime.HandleError(fmt.Errorf("Couldn't get key for %v %#v: %v", rsc.Kind, rs, err))
		return nil
	}
    //如果差值小于0，需要创建新的pod，通过slowStartBatch批量创建pod，返回成功创建的数量
	if diff < 0 {
		diff *= -1
		if diff > rsc.burstReplicas {
			diff = rsc.burstReplicas
		}
	
		rsc.expectations.ExpectCreations(rsKey, diff)
		klog.V(2).Infof("Too few replicas for %v %s/%s, need %d, creating %d", rsc.Kind, rs.Namespace, rs.Name, *(rs.Spec.Replicas), diff)
		
        //通过slowStartBatch批量创建pod，返回成功创建的数量
		successfulCreations, err := slowStartBatch(diff, controller.SlowStartInitialBatchSize, func() error {
			err := rsc.podControl.CreatePodsWithControllerRef(rs.Namespace, &rs.Spec.Template, rs, metav1.NewControllerRef(rs, rsc.GroupVersionKind))
			return err
		})

		//对比差异，看还有缺多个pod没有创建
		if skippedPods := diff - successfulCreations; skippedPods > 0 {
			klog.V(2).Infof("Slow-start failure. Skipping creation of %d pods, decrementing expectations for %v %v/%v", skippedPods, rsc.Kind, rs.Namespace, rs.Name)
			for i := 0; i < skippedPods; i++ {
				// Decrement the expected number of creates because the informer won't observe this pod
				rsc.expectations.CreationObserved(rsKey)
			}
		}
		return err
	} else if diff > 0 {
        //如果diff 大于0，则说明需要进行删除pod
		if diff > rsc.burstReplicas {
			diff = rsc.burstReplicas
		}
		klog.V(2).Infof("Too many replicas for %v %s/%s, need %d, deleting %d", rsc.Kind, rs.Namespace, rs.Name, *(rs.Spec.Replicas), diff)

		// 通过一定的策略返回需要删除的pod,	 not-ready < ready, unscheduled < scheduled, and pending < running. 
		podsToDelete := getPodsToDelete(filteredPods, diff)
	
		rsc.expectations.ExpectDeletions(rsKey, getPodKeys(podsToDelete))

		errCh := make(chan error, diff)
		var wg sync.WaitGroup
		wg.Add(diff)
        //执行删除操作
		for _, pod := range podsToDelete {
			go func(targetPod *v1.Pod) {
				defer wg.Done()
				if err := rsc.podControl.DeletePod(rs.Namespace, targetPod.Name, rs); err != nil {
					// Decrement the expected number of deletes because the informer won't observe this deletion
					podKey := controller.PodKey(targetPod)
					klog.V(2).Infof("Failed to delete %v, decrementing expectations for %v %s/%s", podKey, rsc.Kind, rs.Namespace, rs.Name)
					rsc.expectations.DeletionObserved(rsKey, podKey)
					errCh <- err
				}
			}(pod)
		}
		wg.Wait()

		select {
        //如果某个一个goroutine有错误，则返回错误
		case err := <-errCh:
			// all errors have been reported before and they're likely to be the same, so we'll only return the first one we hit.
			if err != nil {
				return err
			}
		default:
		}
	}

	return nil
}
```

### 总结

目前为止，我们已经分析了controller manager是如何管理多个controller的，并且我们还分析了deployment controller和 replicaset controller的实现逻辑，可以看到controller整个框架逻辑是基本一致的，不同controller主要的不同就是
监听不同资源的add,update, delete，将相应的key添加到informer queue中，然后调用controller各自核心逻辑syncHanlder来实现谐调，最终使得集群应用达到用户期望。

比如大部分contoller逻辑都是如此，流程就是注册name+startcontrollerhandler到 controller manager中，比如`controllers["replicaset"] = startReplicaSetController`
controller manager 会循环启动所有已经注册的name+startcontrollerhandler，对于每一个controller中的具体逻辑，又是通过NewxxxController.Run()的方式执行，
NewxxxCtronller是为了初始化controller，指定controller中需要监听的资源(比如，replicasetcontroller 他需要监听replicaset，pod这个两个资源的变化，而deploymentcontroller 它需要监听deployment,replicaset,pod这三个资源的变化, 具体变化包括add,update,delete 
这三种操作，每个操作可以写相应的逻辑来更新informer catche，修改完成后将key入队)
初始化完资源后，调用Run方法，启动controller，此时Controller会启动指定数量个worker goroutine来从informer队列中拿到key，获取informer catche中的资源和实际集群中的资源进行比较，
此时就进入了每个worker goroutine的synchandler逻辑当中。synchanlder就是核心谐调逻辑。

而replicaset 中synchandler的核心逻辑就是`manageReplicas`，通过不断的对比期望pod，和实际pod差异来添加，或者删除pod。

接下来我会分析kubescheduler源码，欢迎阅读。[]()