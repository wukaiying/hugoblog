---
title: "K8S Controller Manager 源码分析"
date: 2020-04-19T21:08:44+08:00
categories: ["k8s"]
keywords: ["k8s","controller"]  
description: "k8s controller manager 源码分析" 
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

### 前言
 
Controller Manager内部包含Replication Controller、Node Controller、 ResourceQuota Controller、Namespace Controller、ServiceAccount Controller、Token Controller、Service Controller及Endpoint Controller这8种Controller，
每个Controller，它们通过API Server提供的（List-Watch）接口实时监控集群中特定资源的状态变化，当发生各种故障 导致某资源对象的状态发生变化时，Controller会尝试将其状态调整为期望的状态。

### 代码总体解析

#### 入口
```go
# cmd/kube-controller-manager/controller-manager.go

func main() {
	rand.Seed(time.Now().UnixNano())
	command := app.NewControllerManagerCommand()
	logs.InitLogs()
	defer logs.FlushLogs()

	if err := command.Execute(); err != nil {
		os.Exit(1)
	}
}
```

进入`NewControllerManagerCommand` 函数，可以看到新建`ControllerManager`需要两步
启动参数配置，及`Run`函数，进入`Run`函数。
```go
func NewControllerManagerCommand() *cobra.Command {
	s, err := options.NewKubeControllerManagerOptions()
	if err != nil {
		klog.Fatalf("unable to initialize command options: %v", err)
	}
	cmd := &cobra.Command{
		Use: "kube-controller-manager",
		Run: func(cmd *cobra.Command, args []string) {
			verflag.PrintAndExitIfRequested()
			utilflag.PrintFlags(cmd.Flags())
			c, err := s.Config(KnownControllers(), ControllersDisabledByDefault.List())
			if err != nil {
				fmt.Fprintf(os.Stderr, "%v\n", err)
				os.Exit(1)
			}
			if err := Run(c.Complete(), wait.NeverStop); err != nil {
				fmt.Fprintf(os.Stderr, "%v\n", err)
				os.Exit(1)
			}
		},
	}
	return cmd
}
```

#### Run Function 启动流程

Run 函数中主要包含两个步骤，`StartControllers`和`leaderelection`。Kube Controller Manager 既可以单实例启动，也可以多实例启动。 如果为了保证HA一般都会启动多个Controller Manager，通过选举leader的方式来选举一个master来对外提供服务。
关于选举的详细信息可以看我的另外一篇文章[Kubernetes中leaderelection实现组件高可用](https://wukaiying.github.io/k8s/my-first-post/)
```go
// Run runs the KubeControllerManagerOptions.  This should never exit.
func Run(c *config.CompletedConfig, stopCh <-chan struct{}) error {
	run := func(ctx context.Context) {
		controllerContext, err := CreateControllerContext(c, rootClientBuilder, clientBuilder, ctx.Done())
		if err != nil {
			klog.Fatalf("error building controller context: %v", err)
		}
		saTokenControllerInitFunc := serviceAccountTokenControllerStarter{rootClientBuilder: rootClientBuilder}.startServiceAccountTokenController

		if err := StartControllers(controllerContext, saTokenControllerInitFunc, NewControllerInitializers(controllerContext.LoopMode), unsecuredMux); err != nil {
			klog.Fatalf("error starting controllers: %v", err)
		}

		controllerContext.InformerFactory.Start(controllerContext.Stop)
		controllerContext.ObjectOrMetadataInformerFactory.Start(controllerContext.Stop)
		close(controllerContext.InformersStarted)

		select {}
	}

	if !c.ComponentConfig.Generic.LeaderElection.LeaderElect {
		run(context.TODO())
		panic("unreachable")
	}

	id, err := os.Hostname()
	if err != nil {
		return err
	}

	// add a uniquifier so that two processes on the same host don't accidentally both become active
	id = id + "_" + string(uuid.NewUUID())

	rl, err := resourcelock.New(c.ComponentConfig.Generic.LeaderElection.ResourceLock,
		c.ComponentConfig.Generic.LeaderElection.ResourceNamespace,
		c.ComponentConfig.Generic.LeaderElection.ResourceName,
		c.LeaderElectionClient.CoreV1(),
		c.LeaderElectionClient.CoordinationV1(),
		resourcelock.ResourceLockConfig{
			Identity:      id,
			EventRecorder: c.EventRecorder,
		})
	if err != nil {
		klog.Fatalf("error creating lock: %v", err)
	}

	leaderelection.RunOrDie(context.TODO(), leaderelection.LeaderElectionConfig{
		Lock:          rl,
		LeaseDuration: c.ComponentConfig.Generic.LeaderElection.LeaseDuration.Duration,
		RenewDeadline: c.ComponentConfig.Generic.LeaderElection.RenewDeadline.Duration,
		RetryPeriod:   c.ComponentConfig.Generic.LeaderElection.RetryPeriod.Duration,
		Callbacks: leaderelection.LeaderCallbacks{
			OnStartedLeading: run,
			OnStoppedLeading: func() {
				klog.Fatalf("leaderelection lost")
			},
		},
		WatchDog: electionChecker,
		Name:     "kube-controller-manager",
	})
	panic("unreachable")
}
```

#### StartControllers

选主完毕后，就需要真正启动controller了，我们来看一下启动controller 的代码

主要逻辑遍历所有的controllers，controller类型为`controllers map[string]InitFunc`,包括controller名称，及其对应的init 方法。
然后启动每个controller 的Init Function。

```go
// StartControllers starts a set of controllers with a specified ControllerContext
func StartControllers(ctx ControllerContext, startSATokenController InitFunc, controllers map[string]InitFunc, unsecuredMux *mux.PathRecorderMux) error {
	// Always start the SA token controller first using a full-power client, since it needs to mint tokens for the rest
	// If this fails, just return here and fail since other controllers won't be able to get credentials.
	if _, _, err := startSATokenController(ctx); err != nil {
		return err
	}

	// Initialize the cloud provider with a reference to the clientBuilder only after token controller
	// has started in case the cloud provider uses the client builder.
	if ctx.Cloud != nil {
		ctx.Cloud.Initialize(ctx.ClientBuilder, ctx.Stop)
	}

	for controllerName, initFn := range controllers {
		if !ctx.IsControllerEnabled(controllerName) {
			klog.Warningf("%q is disabled", controllerName)
			continue
		}

		time.Sleep(wait.Jitter(ctx.ComponentConfig.Generic.ControllerStartInterval.Duration, ControllerStartJitter))

		klog.V(1).Infof("Starting %q", controllerName)
		debugHandler, started, err := initFn(ctx)
		if err != nil {
			klog.Errorf("Error starting %q", controllerName)
			return err
		}
		if !started {
			klog.Warningf("Skipping %q", controllerName)
			continue
		}
		if debugHandler != nil && unsecuredMux != nil {
			basePath := "/debug/controllers/" + controllerName
			unsecuredMux.UnlistedHandle(basePath, http.StripPrefix(basePath, debugHandler))
			unsecuredMux.UnlistedHandlePrefix(basePath+"/", http.StripPrefix(basePath, debugHandler))
		}
		klog.Infof("Started %q", controllerName)
	}
	return nil
}
```

来看下有多少默认提供的controller

```go
func NewControllerInitializers(loopMode ControllerLoopMode) map[string]InitFunc {
	controllers := map[string]InitFunc{}
	controllers["endpoint"] = startEndpointController
	controllers["endpointslice"] = startEndpointSliceController
	controllers["replicationcontroller"] = startReplicationController
	controllers["podgc"] = startPodGCController
	controllers["resourcequota"] = startResourceQuotaController
	controllers["namespace"] = startNamespaceController
	controllers["serviceaccount"] = startServiceAccountController
	controllers["garbagecollector"] = startGarbageCollectorController
	controllers["daemonset"] = startDaemonSetController
	controllers["job"] = startJobController
	controllers["deployment"] = startDeploymentController
	controllers["replicaset"] = startReplicaSetController
	controllers["horizontalpodautoscaling"] = startHPAController
	controllers["disruption"] = startDisruptionController
	controllers["statefulset"] = startStatefulSetController
	controllers["cronjob"] = startCronJobController
	controllers["csrsigning"] = startCSRSigningController
	controllers["csrapproving"] = startCSRApprovingController
	controllers["csrcleaner"] = startCSRCleanerController
	controllers["ttl"] = startTTLController
	controllers["bootstrapsigner"] = startBootstrapSignerController
	controllers["tokencleaner"] = startTokenCleanerController
	controllers["nodeipam"] = startNodeIpamController
	controllers["nodelifecycle"] = startNodeLifecycleController
	if loopMode == IncludeCloudLoops {
		controllers["service"] = startServiceController
		controllers["route"] = startRouteController
		controllers["cloud-node-lifecycle"] = startCloudNodeLifecycleController
		// TODO: volume controller into the IncludeCloudLoops only set.
	}
	controllers["persistentvolume-binder"] = startPersistentVolumeBinderController
	controllers["attachdetach"] = startAttachDetachController
	controllers["persistentvolume-expander"] = startVolumeExpandController
	controllers["clusterrole-aggregation"] = startClusterRoleAggregrationController
	controllers["pvc-protection"] = startPVCProtectionController
	controllers["pv-protection"] = startPVProtectionController
	controllers["ttl-after-finished"] = startTTLAfterFinishedController
	controllers["root-ca-cert-publisher"] = startRootCACertPublisher

	return controllers
}
```
总结来说，controller manager是所有k8s 资源controller的管控中心，包含的主要逻辑就是高可用选择leader，然后
遍历所有的controllers,对每一个controller进行初始化并启动，也就是说只要集群启动，所有的controller也会都启动起来，开始list-watch api server，来准备创建资源。


### deployment controller

下一节我们看一下deployment controller的详细逻辑：
`deployment.NewDeploymentController`中使用informer来监听deployment，relicaset,pod资源的变化。

```go
//pkg/controller/deployment/deployment_controller.go

func startDeploymentController(ctx ControllerContext) (http.Handler, bool, error) {
	if !ctx.AvailableResources[schema.GroupVersionResource{Group: "apps", Version: "v1", Resource: "deployments"}] {
		return nil, false, nil
	}
	dc, err := deployment.NewDeploymentController(
		ctx.InformerFactory.Apps().V1().Deployments(),
		ctx.InformerFactory.Apps().V1().ReplicaSets(),
		ctx.InformerFactory.Core().V1().Pods(),
		ctx.ClientBuilder.ClientOrDie("deployment-controller"),
	)
	if err != nil {
		return nil, true, fmt.Errorf("error creating Deployment controller: %v", err)
	}
    //start deployment controller
	go dc.Run(int(ctx.ComponentConfig.DeploymentController.ConcurrentDeploymentSyncs), ctx.Stop)
	return nil, true, nil
}
```

#### NewDeploymentController
可以看到DeploymentController,分别监听了deployment，replicaset, pod资源的添加，删除，更新方法。并使用informer的list方法
来获取集群资源，减少与api server交互。
```go
//pkg/controller/deployment/deployment_controller.go:100

// NewDeploymentController creates a new DeploymentController.
func NewDeploymentController(dInformer appsinformers.DeploymentInformer, rsInformer appsinformers.ReplicaSetInformer, podInformer coreinformers.PodInformer, client clientset.Interface) (*DeploymentController, error) {
	dc := &DeploymentController{
		client:        client,
		eventRecorder: eventBroadcaster.NewRecorder(scheme.Scheme, v1.EventSource{Component: "deployment-controller"}),
		queue:         workqueue.NewNamedRateLimitingQueue(workqueue.DefaultControllerRateLimiter(), "deployment"),
	}
	dc.rsControl = controller.RealRSControl{
		KubeClient: client,
		Recorder:   dc.eventRecorder,
	}

	dInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc:    dc.addDeployment,
		UpdateFunc: dc.updateDeployment,
		// This will enter the sync loop and no-op, because the deployment has been deleted from the store.
		DeleteFunc: dc.deleteDeployment,
	})
	rsInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc:    dc.addReplicaSet,
		UpdateFunc: dc.updateReplicaSet,
		DeleteFunc: dc.deleteReplicaSet,
	})
	podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		DeleteFunc: dc.deletePod,
	})

	dc.syncHandler = dc.syncDeployment
	dc.enqueueDeployment = dc.enqueue

	dc.dLister = dInformer.Lister()
	dc.rsLister = rsInformer.Lister()
	dc.podLister = podInformer.Lister()
	dc.dListerSynced = dInformer.Informer().HasSynced
	dc.rsListerSynced = rsInformer.Informer().HasSynced
	dc.podListerSynced = podInformer.Informer().HasSynced
	return dc, nil
}
```

然后我们再看下deployment informer中enventhandler的实现，主要逻辑就是通过List接口，得到add,update,delelete得到提交的deployment资源，然后通过获取`key, err := controller.KeyFunc(deployment)`,
并将Key放入到queue队列中。

```go
//pkg/controller/deployment/deployment_controller.go:120

dInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc:    dc.addDeployment,
		UpdateFunc: dc.updateDeployment,
		// This will enter the sync loop and no-op, because the deployment has been deleted from the store.
		DeleteFunc: dc.deleteDeployment,
	})

//adddeployment的实现
//pkg/controller/deployment/deployment_controller.go:120
func (dc *DeploymentController) addDeployment(obj interface{}) {
	d := obj.(*apps.Deployment)
	klog.V(4).Infof("Adding deployment %s", d.Name)
	dc.enqueueDeployment(d)
}

//enqueueDeployment = enqueue，通过传入Deployment来返回一个key(name+namespace)，并添加到队列中
//pkg/controller/deployment/deployment_controller.go:377
func (dc *DeploymentController) enqueue(deployment *apps.Deployment) {
	key, err := controller.KeyFunc(deployment)
	if err != nil {
		utilruntime.HandleError(fmt.Errorf("Couldn't get key for object %#v: %v", deployment, err))
		return
	}

	dc.queue.Add(key)
}
```

再看一下replicaset informer中eventhandler实现

```go
//pkg/controller/deployment/deployment_controller.go:126
rsInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc:    dc.addReplicaSet,
		UpdateFunc: dc.updateReplicaSet,
		DeleteFunc: dc.deleteReplicaSet,
	})

////pkg/controller/deployment/deployment_controller.go:197

// addReplicaSet enqueues the deployment that manages a ReplicaSet when the ReplicaSet is created.
func (dc *DeploymentController) addReplicaSet(obj interface{}) {
	rs := obj.(*apps.ReplicaSet)

	if rs.DeletionTimestamp != nil {
		// On a restart of the controller manager, it's possible for an object to
		// show up in a state that is already pending deletion.
		dc.deleteReplicaSet(rs)
		return
	}

	// If it has a ControllerRef, that's all that matters.
	if controllerRef := metav1.GetControllerOf(rs); controllerRef != nil {
		d := dc.resolveControllerRef(rs.Namespace, controllerRef)
		if d == nil {
			return
		}
		klog.V(4).Infof("ReplicaSet %s added.", rs.Name)
		dc.enqueueDeployment(d)
		return
	}

	// Otherwise, it's an orphan. Get a list of all matching Deployments and sync
	// them to see if anyone wants to adopt it.
	ds := dc.getDeploymentsForReplicaSet(rs)
	if len(ds) == 0 {
		return
	}
	klog.V(4).Infof("Orphan ReplicaSet %s added.", rs.Name)
	for _, d := range ds {
		dc.enqueueDeployment(d)
	}
}
```
通过阅读addReplicaset我们可以发现，当replicaset发生变化时，首先会通过replicaset中的ownnerreference结构体`d := dc.resolveControllerRef(rs.Namespace, controllerRef)`来，找到
与他有关联的deployment，随着rs的add,update,delete，将相应的deploymnet 的key(name+namepace)添加到队列queue中。

最后看一下`podInformer.Informer()`中的方法：
注释描述的很清楚，当deployment对应的pod都不存在时，将该deployment放入队列queue中。
```go
// deletePod will enqueue a Recreate Deployment once all of its pods have stopped running.
func (dc *DeploymentController) deletePod(obj interface{}) {
	pod, ok := obj.(*v1.Pod)

	// When a delete is dropped, the relist will notice a pod in the store not
	// in the list, leading to the insertion of a tombstone object which contains
	// the deleted key/value. Note that this value might be stale. If the Pod
	// changed labels the new deployment will not be woken up till the periodic resync.
	if !ok {
		tombstone, ok := obj.(cache.DeletedFinalStateUnknown)
		if !ok {
			utilruntime.HandleError(fmt.Errorf("Couldn't get object from tombstone %#v", obj))
			return
		}
		pod, ok = tombstone.Obj.(*v1.Pod)
		if !ok {
			utilruntime.HandleError(fmt.Errorf("Tombstone contained object that is not a pod %#v", obj))
			return
		}
	}
	klog.V(4).Infof("Pod %s deleted.", pod.Name)
	if d := dc.getDeploymentForPod(pod); d != nil && d.Spec.Strategy.Type == apps.RecreateDeploymentStrategyType {
		// Sync if this Deployment now has no more Pods.
		rsList, err := util.ListReplicaSets(d, util.RsListFromClient(dc.client.AppsV1()))
		if err != nil {
			return
		}
		podMap, err := dc.getPodMapForDeployment(d, rsList)
		if err != nil {
			return
		}
		numPods := 0
		for _, podList := range podMap {
			numPods += len(podList)
		}
		if numPods == 0 {
			dc.enqueueDeployment(d)
		}
	}
}
```

#### Run deployment controller

上面构建好了deployment controller，接下来看controller的启动函数
`cache.WaitForNamedCacheSync` 扥带Informer cache完成，`go wait.Until(dc.worker, time.Second, stopCh)`则启动指定数量的deployment goroutine 来处理提交的资源，直到处理完成，

```go
// Run begins watching and syncing.
// //pkg/controller/deployment/deployment_controller.go:148
func (dc *DeploymentController) Run(workers int, stopCh <-chan struct{}) {
	defer utilruntime.HandleCrash()
	defer dc.queue.ShutDown()
	
	if !cache.WaitForNamedCacheSync("deployment", stopCh, dc.dListerSynced, dc.rsListerSynced, dc.podListerSynced) {
		return
	}

	for i := 0; i < workers; i++ {
		go wait.Until(dc.worker, time.Second, stopCh)
	}

	<-stopCh
}

// worker runs a worker thread that just dequeues items, processes them, and marks them done.
// It enforces that the syncHandler is never invoked concurrently with the same key.
func (dc *DeploymentController) worker() {
	for dc.processNextWorkItem() {
	}
}
// //pkg/controller/deployment/deployment_controller.go:460
func (dc *DeploymentController) processNextWorkItem() bool {
	key, quit := dc.queue.Get()
	if quit {
		return false
	}
	defer dc.queue.Done(key)

	err := dc.syncHandler(key.(string))
	dc.handleErr(err, key)

	return true
}
```

`dc.worker` 里面描述了worker，也就是deploymentcontroller所做的具体逻辑, 不断地从informer queue中取出key
然后调用`dc.syncHandler(key.(string))`，接下来看下syncHandler中的逻辑：

```go
// syncDeployment will sync the deployment with the given key.
// This function is not meant to be invoked concurrently with the same key.
func (dc *DeploymentController) syncDeployment(key string) error {
	startTime := time.Now()
	klog.V(4).Infof("Started syncing deployment %q (%v)", key, startTime)
	defer func() {
		klog.V(4).Infof("Finished syncing deployment %q (%v)", key, time.Since(startTime))
	}()

	namespace, name, err := cache.SplitMetaNamespaceKey(key)
	if err != nil {
		return err
	}
	deployment, err := dc.dLister.Deployments(namespace).Get(name)
	if errors.IsNotFound(err) {
		klog.V(2).Infof("Deployment %v has been deleted", key)
		return nil
	}
	if err != nil {
		return err
	}

	// Deep-copy otherwise we are mutating our cache.
	// TODO: Deep-copy only when needed.
	d := deployment.DeepCopy()

	everything := metav1.LabelSelector{}
	if reflect.DeepEqual(d.Spec.Selector, &everything) {
		dc.eventRecorder.Eventf(d, v1.EventTypeWarning, "SelectingAll", "This deployment is selecting all pods. A non-empty selector is required.")
		if d.Status.ObservedGeneration < d.Generation {
			d.Status.ObservedGeneration = d.Generation
			dc.client.AppsV1().Deployments(d.Namespace).UpdateStatus(d)
		}
		return nil
	}

	// List ReplicaSets owned by this Deployment, while reconciling ControllerRef
	// through adoption/orphaning.
	rsList, err := dc.getReplicaSetsForDeployment(d)
	if err != nil {
		return err
	}
	// List all Pods owned by this Deployment, grouped by their ReplicaSet.
	// Current uses of the podMap are:
	//
	// * check if a Pod is labeled correctly with the pod-template-hash label.
	// * check that no old Pods are running in the middle of Recreate Deployments.
	podMap, err := dc.getPodMapForDeployment(d, rsList)
	if err != nil {
		return err
	}

	if d.DeletionTimestamp != nil {
		return dc.syncStatusOnly(d, rsList)
	}

	// Update deployment conditions with an Unknown condition when pausing/resuming
	// a deployment. In this way, we can be sure that we won't timeout when a user
	// resumes a Deployment with a set progressDeadlineSeconds.
	if err = dc.checkPausedConditions(d); err != nil {
		return err
	}

	if d.Spec.Paused {
		return dc.sync(d, rsList)
	}

	// rollback is not re-entrant in case the underlying replica sets are updated with a new
	// revision so we should ensure that we won't proceed to update replica sets until we
	// make sure that the deployment has cleaned up its rollback spec in subsequent enqueues.
	if getRollbackTo(d) != nil {
		return dc.rollback(d, rsList)
	}

	scalingEvent, err := dc.isScalingEvent(d, rsList)
	if err != nil {
		return err
	}
	if scalingEvent {
		return dc.sync(d, rsList)
	}

	switch d.Spec.Strategy.Type {
	case apps.RecreateDeploymentStrategyType:
		return dc.rolloutRecreate(d, rsList, podMap)
	case apps.RollingUpdateDeploymentStrategyType:
		return dc.rolloutRolling(d, rsList)
	}
	return fmt.Errorf("unexpected deployment strategy type: %s", d.Spec.Strategy.Type)
}
```
主要逻辑如下：
1. 从queue中拿到key，通过key得到deployment name和namespace

2. 然后通过`deployment, err := dc.dLister.Deployments(namespace).Get(name)`得到deployment资源

3. 通过deployment得到与之对应的replicaset list.`rsList, err := dc.getReplicaSetsForDeployment(d)` 

4. 然后通过`podMap, err := dc.getPodMapForDeployment(d, rsList)` 来获取deployment对应的 pod

5. 然后通过`getRollbackTo`和`isScalingEvent` 分别判断deployment是否处于这两个状态，如果是这执行对应的操作，直到完成才能进入6.

6. 最后通过Strategy来判断本次deployment 所要做的操作是`RecreateDeploymentStrategyType`还是`RollingUpdateDeploymentStrategyType`。

进入RecreateDeploymentStrategyType 进行分析：

```go
//pkg/controller/deployment/recreate.go:27
// rolloutRecreate implements the logic for recreating a replica set.
func (dc *DeploymentController) rolloutRecreate(d *apps.Deployment, rsList []*apps.ReplicaSet, podMap map[types.UID][]*v1.Pod) error {
	// Don't create a new RS if not already existed, so that we avoid scaling up before scaling down.
	newRS, oldRSs, err := dc.getAllReplicaSetsAndSyncRevision(d, rsList, false)
	if err != nil {
		return err
	}
	allRSs := append(oldRSs, newRS)
	activeOldRSs := controller.FilterActiveReplicaSets(oldRSs)

	// scale down old replica sets.
	scaledDown, err := dc.scaleDownOldReplicaSetsForRecreate(activeOldRSs, d)
	if err != nil {
		return err
	}
	if scaledDown {
		// Update DeploymentStatus.
		return dc.syncRolloutStatus(allRSs, newRS, d)
	}

	// Do not process a deployment when it has old pods running.
	if oldPodsRunning(newRS, oldRSs, podMap) {
		return dc.syncRolloutStatus(allRSs, newRS, d)
	}

	// If we need to create a new RS, create it now.
	if newRS == nil {
		newRS, oldRSs, err = dc.getAllReplicaSetsAndSyncRevision(d, rsList, true)
		if err != nil {
			return err
		}
		allRSs = append(oldRSs, newRS)
	}

	// scale up new replica set.
	if _, err := dc.scaleUpNewReplicaSetForRecreate(newRS, d); err != nil {
		return err
	}

	if util.DeploymentComplete(d, &d.Status) {
		if err := dc.cleanupDeployment(oldRSs, d); err != nil {
			return err
		}
	}

	// Sync deployment status.
	return dc.syncRolloutStatus(allRSs, newRS, d)
}
```
该模块逻辑很明晰：
1. 获取新旧repliaceset list

2. scale down 老的replicaset，将他的replicas设置为0，当然第一次创建的时候是没有老ReplicaSet的

3. 如果第一次创建，那么需要去创建对应的ReplicaSet

4. 创建完毕对应的ReplicaSet后 扩容ReplicaSet 到对应的值

5. 等待新建的创建完毕，清理老的ReplcaiSet

6. 更新Deployment Status

进入RollingUpdateDeploymentStrategyType 进行分析：

```go
// rolloutRolling implements the logic for rolling a new replica set.
func (dc *DeploymentController) rolloutRolling(d *apps.Deployment, rsList []*apps.ReplicaSet) error {
	newRS, oldRSs, err := dc.getAllReplicaSetsAndSyncRevision(d, rsList, true)
	if err != nil {
		return err
	}
	allRSs := append(oldRSs, newRS)

	// Scale up, if we can.
	scaledUp, err := dc.reconcileNewReplicaSet(allRSs, newRS, d)
	if err != nil {
		return err
	}
	if scaledUp {
		// Update DeploymentStatus
		return dc.syncRolloutStatus(allRSs, newRS, d)
	}

	// Scale down, if we can.
	scaledDown, err := dc.reconcileOldReplicaSets(allRSs, controller.FilterActiveReplicaSets(oldRSs), newRS, d)
	if err != nil {
		return err
	}
	if scaledDown {
		// Update DeploymentStatus
		return dc.syncRolloutStatus(allRSs, newRS, d)
	}

	if deploymentutil.DeploymentComplete(d, &d.Status) {
		if err := dc.cleanupDeployment(oldRSs, d); err != nil {
			return err
		}
	}

	// Sync deployment status
	return dc.syncRolloutStatus(allRSs, newRS, d)
}
```

1.获取新旧replicaset，如果没有新的则创建

2.scale up 新的replicaset，将其replicas设置为指定值比如3

3.scale down 老的replicaset，将其replicas设置为另外一个指定值比如2

4.通过循环直到new replicaset pod 副本数达到10，old replicaset pod 副本数为0，完成

4.更新deployment 状态

下面来看下rollingupdate 算法层面实现：

假设我们创建一个 nginx-deployment 有10 个副本，等 10 个 pod 都启动完成后如下所示：
```
$ kubectl create -f nginx-dep.yaml

$ kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-68b649bd8b   10        10        10      72m
```
然后更新 nginx-deployment 的镜像，默认使用滚动更新的方式：
```
$ kubectl set image deploy/nginx-deployment nginx-deployment=nginx:1.9.3
```
此时通过源码可知会计算该 deployment 的 maxSurge、maxUnavailable 和 maxAvailable 的值，分别为 3、2 和 13，计算方法如下所示：
```
// 向上取整为 3
maxSurge = replicas * deployment.spec.strategy.rollingUpdate.maxSurge(25%)= 2.5

// 向下取整为 2
maxUnavailable = replicas * deployment.spec.strategy.rollingUpdate.maxUnavailable(25%)= 2.5

maxAvailable = replicas(10) + MaxSurge（3） = 13
```
此时deployment controller会创建一个新的newRS，将其replicas设置为3，然后开始创建pod副本，当所有 rs 的 replicas 已经达到最大值 10 + 3 = 13时，

此时oldRS需要减少maxUnavailable=2个pod, 然后newRS pod增加2个，直到newRs Pod为10，oldRs pod为0，完成rollingupdate。


### 总结

deployment 的本质是控制 replicaSet，replicaSet 会控制 pod，然后由 controller 驱动各个对象达到期望状态。

![deployment replicaset pod关系](https://pic.images.ac.cn/image/5e9ea587ab857.html)

DeploymentController 是 Deployment 资源的控制器，其通过 DeploymentInformer(add事件,update事件,delete事件)、ReplicaSetInformer(add事件 update事件 ,delete事件)、PodInformer(delete事件) 
来执行循环Loop操作。

当deployment recreate时，deployment controller会调用replicaset controller逻辑，先将老的replicaset 副本数量设置为0，再启动一个新的replicasset，来
启动指定数量的pod，最后更新deployment状态

当deployment 发送rollingupdate时，先启动新的replicaset，将replicas设置为指定数量，
然后再将老的replicaset，replicas设置为0,最后更新deployment状态

最后，deployment controller并不会直接控制pod的创建和删除，而是通过新建replicaset,并修改副本个数的方式来实现pod的更新与删除。
接下来我们会看replicaset是如何控制pod的创建和删除，更新的。

[Replica Set Controller 源码分析](https://wukaiying.github.io/k8s/replicaset-source-code-read/)

### 参考
https://yq.aliyun.com/articles/688622
https://blog.tianfeiyu.com/source-code-reading-notes/kubernetes/deployment_controller.html