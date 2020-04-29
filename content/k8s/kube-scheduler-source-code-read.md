---
title: "K8S Scheduler 源码分析"
date: 2020-04-22T13:56:37+08:00
draft: false
categories: ["k8s"]
keywords: ["k8s","scheduler"]  
description: "k8s scheduler 源码分析" 
summaryLength: 100
---

### 版本环境
- kubernetes版本：kubernetes:v1.16.0

- go环境：go version go1.13.4 darwin/amd64

### 前言

kube-scheduler 是master节点中重要的组件之一，它同watch接口监听api server中需要调度的pod，然后通过其预选，优选算法来确定pod需要被调度到哪个节点上去。

看一下官方对scheduler 调度流程的描述：[The Kubernetes Scheduler](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-scheduling/scheduler.md)

主要步骤如下：
- 首先通过预选阶段过滤掉不合适的节点，例如根据podSpec中对系统资源的需求，会计算出所有节点剩余的资源是否可以满足pod需求，如果不满足则该节点会被抛弃。
- 其次通过预选阶段的节点，会再次进行优选。比如在满足pod资源需求的基础上，会优选出当前负载最小节点，保证每个集群节点资源被均衡使用。
- 最后选择打分最高的节点作为目标节点，如果得分相同则会随机选择一个，进行调度。
```
For given pod:

    +---------------------------------------------+
    |               Schedulable nodes:            |
    |                                             |
    | +--------+    +--------+      +--------+    |
    | | node 1 |    | node 2 |      | node 3 |    |
    | +--------+    +--------+      +--------+    |
    |                                             |
    +-------------------+-------------------------+
                        |
                        |
                        v
    +-------------------+-------------------------+

    Pred. filters: node 3 doesn't have enough resource

    +-------------------+-------------------------+
                        |
                        |
                        v
    +-------------------+-------------------------+
    |             remaining nodes:                |
    |   +--------+                 +--------+     |
    |   | node 1 |                 | node 2 |     |
    |   +--------+                 +--------+     |
    |                                             |
    +-------------------+-------------------------+
                        |
                        |
                        v
    +-------------------+-------------------------+

    Priority function:    node 1: p=2
                          node 2: p=5

    +-------------------+-------------------------+
                        |
                        |
                        v
            select max{node priority} = node 2
```

### 入口
scheduler使用cobra创建命令行客户端，进入`NewSchedulerCommand`

```go
//cmd/kube-scheduler/scheduler.go:32
func main() {
	rand.Seed(time.Now().UnixNano())
	command := app.NewSchedulerCommand()
	pflag.CommandLine.SetNormalizeFunc(cliflag.WordSepNormalizeFunc)
	// utilflag.InitFlags()
	logs.InitLogs()
	defer logs.FlushLogs()
	if err := command.Execute(); err != nil {
		os.Exit(1)
	}
}
```

#### NewSchedulerCommand

我们可以看到该方法通过接受用户传递进来的scheduler参数来通过`runCommand`方法来启动scheduler，下面进入`runCommand`
```go
//cmd/kube-scheduler/app/server.go:69
func NewSchedulerCommand(registryOptions ...Option) *cobra.Command {
	opts, err := options.NewOptions()
	if err != nil {
		klog.Fatalf("unable to initialize command options: %v", err)
	}

	cmd := &cobra.Command{
		Use: "kube-scheduler",
		Run: func(cmd *cobra.Command, args []string) {
			if err := runCommand(cmd, args, opts, registryOptions...); err != nil {
				fmt.Fprintf(os.Stderr, "%v\n", err)
				os.Exit(1)
			}
		},
	}
	return cmd
}
```
#### runCommand

runCommand方法主要作用是校验scheduler 配置参数，生成scheduler配置文件，并通过Run方法来启动scheduler。主要流程如下：

- 校验通过cobra传递进来的配置参数
- 生成scheduler 配置项
- 不全得到完整的scheduler启动参数结构体
- 将完整的启动参数结构体传入Run函数，然后启动scheduler

接下来我们进入Run方法来看，scheduler具体如何启动。

```go
//cmd/kube-scheduler/app/server.go:116
// runCommand runs the scheduler.
func runCommand(cmd *cobra.Command, args []string, opts *options.Options, registryOptions ...Option) error {
	
    //校验通过cobra传递进来的配置参数
    verflag.PrintAndExitIfRequested()
	utilflag.PrintFlags(cmd.Flags())

	if len(args) != 0 {
		fmt.Fprint(os.Stderr, "arguments are not supported\n")
	}

	if errs := opts.Validate(); len(errs) > 0 {
		fmt.Fprintf(os.Stderr, "%v\n", utilerrors.NewAggregate(errs))
		os.Exit(1)
	}

	if len(opts.WriteConfigTo) > 0 {
		if err := options.WriteConfigFile(opts.WriteConfigTo, &opts.ComponentConfig); err != nil {
			fmt.Fprintf(os.Stderr, "%v\n", err)
			os.Exit(1)
		}
		klog.Infof("Wrote configuration to: %s\n", opts.WriteConfigTo)
	}
    //生成scheduler 配置项
	c, err := opts.Config()
	if err != nil {
		fmt.Fprintf(os.Stderr, "%v\n", err)
		os.Exit(1)
	}

	stopCh := make(chan struct{})
	// Get the completed config
    //得到完整的scheduler启动参数结构体
	cc := c.Complete()

	// To help debugging, immediately log version
	klog.Infof("Version: %+v", version.Get())

	// Apply algorithms based on feature gates.
	// TODO: make configurable?
	algorithmprovider.ApplyFeatureGates()

	// Configz registration.
	if cz, err := configz.New("componentconfig"); err == nil {
		cz.Set(cc.ComponentConfig)
	} else {
		return fmt.Errorf("unable to register configz: %s", err)
	}

    //将完整的启动参数结构体传入Run函数，然后启动scheduler
	return Run(cc, stopCh, registryOptions...)
}
```

#### Run 启动scheduler

```go
//cmd/kube-scheduler/app/server.go:165
// Run executes the scheduler based on the given configuration. It only return on error or when stopCh is closed.
func Run(cc schedulerserverconfig.CompletedConfig, stopCh <-chan struct{}, registryOptions ...Option) error {
	// To help debugging, immediately log version
	klog.V(1).Infof("Starting Kubernetes Scheduler version %+v", version.Get())

	registry := framework.NewRegistry()
	for _, option := range registryOptions {
		if err := option(registry); err != nil {
			return err
		}
	}

	// 初始化 event clients，记录调度事件
	if _, err := cc.Client.Discovery().ServerResourcesForGroupVersion(eventsv1beta1.SchemeGroupVersion.String()); err == nil {
		cc.Broadcaster = events.NewBroadcaster(&events.EventSinkImpl{Interface: cc.EventClient.Events("")})
		cc.Recorder = cc.Broadcaster.NewRecorder(scheme.Scheme, cc.ComponentConfig.SchedulerName)
	} else {
		recorder := cc.CoreBroadcaster.NewRecorder(scheme.Scheme, v1.EventSource{Component: cc.ComponentConfig.SchedulerName})
		cc.Recorder = record.NewEventRecorderAdapter(recorder)
	}

	//新建scheduler
	sched, err := scheduler.New(cc.Client,
		cc.InformerFactory.Core().V1().Nodes(),
		cc.PodInformer,
		cc.InformerFactory.Core().V1().PersistentVolumes(),
		cc.InformerFactory.Core().V1().PersistentVolumeClaims(),
		cc.InformerFactory.Core().V1().ReplicationControllers(),
		cc.InformerFactory.Apps().V1().ReplicaSets(),
		cc.InformerFactory.Apps().V1().StatefulSets(),
		cc.InformerFactory.Core().V1().Services(),
		cc.InformerFactory.Policy().V1beta1().PodDisruptionBudgets(),
		cc.InformerFactory.Storage().V1().StorageClasses(),
		cc.InformerFactory.Storage().V1beta1().CSINodes(),
		cc.Recorder,
		cc.ComponentConfig.AlgorithmSource,
		stopCh,
		registry,
		cc.ComponentConfig.Plugins,
		cc.ComponentConfig.PluginConfig,
		scheduler.WithName(cc.ComponentConfig.SchedulerName),
		scheduler.WithHardPodAffinitySymmetricWeight(cc.ComponentConfig.HardPodAffinitySymmetricWeight),
		scheduler.WithPreemptionDisabled(cc.ComponentConfig.DisablePreemption),
		scheduler.WithPercentageOfNodesToScore(cc.ComponentConfig.PercentageOfNodesToScore),
		scheduler.WithBindTimeoutSeconds(*cc.ComponentConfig.BindTimeoutSeconds))
	if err != nil {
		return err
	}

	// 启动事件广播
	if cc.Broadcaster != nil && cc.EventClient != nil {
		cc.Broadcaster.StartRecordingToSink(stopCh)
	}
	if cc.CoreBroadcaster != nil && cc.CoreEventClient != nil {
		cc.CoreBroadcaster.StartRecordingToSink(&corev1.EventSinkImpl{Interface: cc.CoreEventClient.Events("")})
	}

	// 启动podinformer，监听pod事件
	go cc.PodInformer.Informer().Run(stopCh)
	cc.InformerFactory.Start(stopCh)

	// 调度之前等待informer初始化cache完成
	cc.InformerFactory.WaitForCacheSync(stopCh)

	// Prepare a reusable runCommand function.
	run := func(ctx context.Context) {
		sched.Run()
		<-ctx.Done()
	}

	ctx, cancel := context.WithCancel(context.TODO()) // TODO once Run() accepts a context, it should be used here
	defer cancel()

	go func() {
		select {
		case <-stopCh:
			cancel()
		case <-ctx.Done():
		}
	}()

	// 选举出leader
	if cc.LeaderElection != nil {
		cc.LeaderElection.Callbacks = leaderelection.LeaderCallbacks{
			OnStartedLeading: run,
			OnStoppedLeading: func() {
				klog.Fatalf("leaderelection lost")
			},
		}
		leaderElector, err := leaderelection.NewLeaderElector(*cc.LeaderElection)
		if err != nil {
			return fmt.Errorf("couldn't create leader elector: %v", err)
		}

		leaderElector.Run(ctx)

		return fmt.Errorf("lost lease")
	}

	// 选举完成后执行sched.Run()
	run(ctx)
	return fmt.Errorf("finished without leader elect")
}
```

接下来我们进入`sched.Run()`方法来看，它启动一个循环逻辑，来执行`sched.scheduleOne`
```go
//pkg/scheduler/scheduler.go:315
func (sched *Scheduler) Run() {
	if !sched.WaitForCacheSync() {
		return
	}
	go wait.Until(sched.scheduleOne, 0, sched.StopEverything)
}
```

现在我们来看下`sched.scheduleOne`的实现，通过goroutine 启动的scheduleOne，每个协程一次只会为一个pod执行调度计算。
主要流程如下：

- 从待调度队列中取出一个pod，这块背后是由SchedulingQueue该模块实现，暂不进行深入分析
- 如果pod被标记为待删除，则不进行调度
- 加载调度策略，通过sched.schedule来对pod进行调度，如果调度成功会得到一个node节点名称（此时已经经过预选和优选完成）
- 如果调度失败，查看返回的错误中，该pod是否启动了抢占策略，如果没有则什么也不做，pod重新入队，重新调度。
- 执行绑定操作，也就是将pod.Spec.NodeName设置为选出来的node

```go
//pkg/scheduler/scheduler.go:517
// scheduleOne does the entire scheduling workflow for a single pod.  It is serialized on the scheduling algorithm's host fitting.
func (sched *Scheduler) scheduleOne() {
	fwk := sched.Framework

    //从待调度队列中取出一个pod，这块背后是由SchedulingQueue该模块实现，暂不进行深入分析
	pod := sched.NextPod()
	// pod could be nil when schedulerQueue is closed
	if pod == nil {
		return
	}
    //如果pod被标记为待删除，则不进行调度
	if pod.DeletionTimestamp != nil {
		sched.Recorder.Eventf(pod, nil, v1.EventTypeWarning, "FailedScheduling", "Scheduling", "skip schedule deleting pod: %v/%v", pod.Namespace, pod.Name)
		klog.V(3).Infof("Skip schedule deleting pod: %v/%v", pod.Namespace, pod.Name)
		return
	}

	klog.V(3).Infof("Attempting to schedule pod: %v/%v", pod.Namespace, pod.Name)

	//加载调度策略，通过sched.schedule来对pod进行调度
	start := time.Now()
	pluginContext := framework.NewPluginContext()

    //此时如果err为空的话，scheduleResult中已经得到了目标node，如果err不为空则进行抢占调度
	scheduleResult, err := sched.schedule(pod, pluginContext)
    
    //如果调度失败，查看返回的错误中，该pod是否启动了抢占策略，如果没有则什么也不做，pod重新入队，重新调度。
    //如果启动了抢占策略(这里启动了抢占调度，对应于yaml文件中是为pod绑定了PriorityClass这种资源文件)，提供了一种抢占策略，可以挤掉优先级不高pod，抢先进行调度。
	if err != nil {
		if fitError, ok := err.(*core.FitError); ok {
			if sched.DisablePreemption {
				klog.V(3).Infof("Pod priority feature is not enabled or preemption is disabled by scheduler configuration." +
					" No preemption is performed.")
			} else {
                //执行抢占，并通过过滤算法选择合适的node
				preemptionStartTime := time.Now()
				sched.preempt(pluginContext, fwk, pod, fitError)
			}
			metrics.PodScheduleFailures.Inc()
		} else {
			klog.Errorf("error selecting node for pod: %v", err)
			metrics.PodScheduleErrors.Inc()
		}
		return
	}
	
	assumedPod := pod.DeepCopy()

	//执行绑定操作，也就是将pod.Spec.NodeName设置为选出来的node
	err = sched.assume(assumedPod, scheduleResult.SuggestedHost)
	if err != nil {
		klog.Errorf("error assuming pod: %v", err)
		metrics.PodScheduleErrors.Inc()
		// trigger un-reserve plugins to clean up state associated with the reserved Pod
		fwk.RunUnreservePlugins(pluginContext, assumedPod, scheduleResult.SuggestedHost)
		return
	}
}
```

这里有必要分析下`scheduleResult, err := sched.schedule(pod, pluginContext)`，当err不为空时，说明没有合适的node进行调度，如果用户给pod配置了高调度优先级，则会触发pod抢占调度逻辑，抢占调度的具体应用场景及源码逻辑如下，

首先对应于yaml文件，我们可以这样提高pod的调度优先级
```yaml
# Example PriorityClass
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "This priority class should be used for XYZ service pods only."

# Example Pod spec
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  priorityClassName: high-priority
```
来看下抢占核心逻辑`sched.preempt(pluginContext, fwk, pod, fitError)`的具体实现：
主要逻辑如下：

- 通过默认注册的抢占算法，计算得出最终被执行抢占调度的node、node上需要驱逐的pod等信息
- 可以进行抢占调度的node不为空时，给PriorityQueue队列中添加新元素(调度的pod和目标node的信息)
- 接下来给抢占调度的pod加上指定的NominatedNodeName，也就是设置pod.Status.NominatedNodeName=nodeName
- 删除node上低优先级的pod
- 当抢占执行完成，或者没有合适的node是，都会执行删除pod.Status.NominatedNodeName=nodeName，以免影响下次抢占调度

```go
func (sched *Scheduler) preempt(pluginContext *framework.PluginContext, fwk framework.Framework, preemptor *v1.Pod, scheduleErr error) (string, error) {
	preemptor, err := sched.PodPreemptor.GetUpdatedPod(preemptor)
	if err != nil {
		klog.Errorf("Error getting the updated preemptor pod object: %v", err)
		return "", err
	}
    //  // 通过默认注册的抢占算法，计算得出最终被执行抢占调度的node、node上需要驱逐的pod等信息
	node, victims, nominatedPodsToClear, err := sched.Algorithm.Preempt(pluginContext, preemptor, scheduleErr)
	if err != nil {
		klog.Errorf("Error preempting victims to make room for %v/%v: %v", preemptor.Namespace, preemptor.Name, err)
		return "", err
	}
	var nodeName = ""
    //可以进行抢占调度的node不为空时
	if node != nil {
		nodeName = node.Name
		// Update the scheduling queue with the nominated pod information. Without
		// this, there would be a race condition between the next scheduling cycle
		// and the time the scheduler receives a Pod Update for the nominated pod.
		// 给SchedulingQueue队列中添加抢占调度的pod和目标node的信息
        sched.SchedulingQueue.UpdateNominatedPodForNode(preemptor, nodeName)

		// Make a call to update nominated node name of the pod on the API server.
        //给抢占调度的pod加上指定的NominatedNodeName，也就是设置pod.Status.NominatedNodeName=nodeName
		err = sched.PodPreemptor.SetNominatedNodeName(preemptor, nodeName)
		if err != nil {
			klog.Errorf("Error in preemption process. Cannot set 'NominatedPod' on pod %v/%v: %v", preemptor.Namespace, preemptor.Name, err)
			sched.SchedulingQueue.DeleteNominatedPodIfExists(preemptor)
			return "", err
		}

        //删除nodeName上低优先级的pod
		for _, victim := range victims {
			if err := sched.PodPreemptor.DeletePod(victim); err != nil {
				klog.Errorf("Error preempting pod %v/%v: %v", victim.Namespace, victim.Name, err)
				return "", err
			}
			// If the victim is a WaitingPod, send a reject message to the PermitPlugin
			if waitingPod := fwk.GetWaitingPod(victim.UID); waitingPod != nil {
				waitingPod.Reject("preempted")
			}
			sched.Recorder.Eventf(victim, preemptor, v1.EventTypeNormal, "Preempted", "Preempting", "Preempted by %v/%v on node %v", preemptor.Namespace, preemptor.Name, nodeName)

		}
		metrics.PreemptionVictims.Set(float64(len(victims)))
	}
	// Clearing nominated pods should happen outside of "if node != nil". Node could
	// be nil when a pod with nominated node name is eligible to preempt again,
	// but preemption logic does not find any node for it. In that case Preempt()
	// function of generic_scheduler.go returns the pod itself for removal of
	// the 'NominatedPod' field.
    //这里有两种情况，当可以执行抢占的node不存在，则清除pod中的NominatedNodeName字段。
    //抢占逻辑执行完成后，也需要执行清除pod中的NominatedNodeName字段。
	for _, p := range nominatedPodsToClear {
		rErr := sched.PodPreemptor.RemoveNominatedNodeName(p)
		if rErr != nil {
			klog.Errorf("Cannot remove 'NominatedPod' field of pod: %v", rErr)
			// We do not return as this error is not critical.
		}
	}
	return nodeName, err
}

```

如果`scheduleResult, err := sched.schedule(pod, pluginContext)`，err为空，则说明调度成功，不需要执行抢占，这里面就包含了调度器的预选，优选算法，我们详细分析下
进入schedule
```go
//pkg/scheduler/scheduler.go:339
// schedule implements the scheduling algorithm and returns the suggested result(host,
// evaluated nodes number,feasible nodes number).
func (sched *Scheduler) schedule(pod *v1.Pod, pluginContext *framework.PluginContext) (core.ScheduleResult, error) {
	result, err := sched.Algorithm.Schedule(pod, pluginContext)
	if err != nil {
		pod = pod.DeepCopy()
		sched.recordSchedulingFailure(pod, err, v1.PodReasonUnschedulable, err.Error())
		return core.ScheduleResult{}, err
	}
	return result, err
}
```
发现`sched.Algorithm.Schedule`他是`ScheduleAlgorithm`接口的一个实现，里面包含四个方法，我们来看Schedule的具体实现
```go
///pkg/scheduler/core/generic_scheduler.go:134
type ScheduleAlgorithm interface {
	Schedule(*v1.Pod, *framework.PluginContext) (scheduleResult ScheduleResult, err error)
	
	Preempt(*framework.PluginContext, *v1.Pod, error) (selectedNode *v1.Node, preemptedPods []*v1.Pod, cleanupNominatedPods []*v1.Pod, err error)
	
	Predicates() map[string]predicates.FitPredicate
	
	Prioritizers() []priorities.PriorityConfig
}
```

Schedule的具体实现中包含了预选优选两个阶段，主要步骤如下：

- pod做一些前置过滤
- 获取node数量 numNodes := g.cache.NodeTree().NumNodes()
- 执行预选算法，得到可用的node filteredNodes。filteredNodes = g.findNodesThatFit(pluginContext, pod)
- 如果预选完成没有合适的node,如果只有一个Node,则直接返回该node
- 在优选函数中传入pod信息，和符合条件的node信息，进行优选
- 通过打分机制选择出最合适的node，host= g.selectHost(priorityList)，完成scheduler调度

```go
func (g *genericScheduler) Schedule(pod *v1.Pod, pluginContext *framework.PluginContext) (result ScheduleResult, err error) {
	trace := utiltrace.New("Scheduling", utiltrace.Field{Key: "namespace", Value: pod.Namespace}, utiltrace.Field{Key: "name", Value: pod.Name})
	defer trace.LogIfLong(100 * time.Millisecond)

    //下面这两部对pod做一些前置过滤
	if err := podPassesBasicChecks(pod, g.pvcLister); err != nil {
		return result, err
	}
	preFilterStatus := g.framework.RunPreFilterPlugins(pluginContext, pod)
	if !preFilterStatus.IsSuccess() {
		return result, preFilterStatus.AsError()
	}
    
    //获取node数量
	numNodes := g.cache.NodeTree().NumNodes()
	if numNodes == 0 {
		return result, ErrNoNodesAvailable
	}
	if err := g.snapshot(); err != nil {
		return result, err
	}

	trace.Step("Basic checks done")
	startPredicateEvalTime := time.Now()
    //执行预选算法，得到可用的node filteredNodes。
	filteredNodes, failedPredicateMap, filteredNodesStatuses, err := g.findNodesThatFit(pluginContext, pod)
	if err != nil {
		return result, err
	}
    //如果预选完成没有合适的node,返回error
	if len(filteredNodes) == 0 {
		return result, &FitError{
			Pod:                   pod,
			NumAllNodes:           numNodes,
			FailedPredicates:      failedPredicateMap,
			FilteredNodesStatuses: filteredNodesStatuses,
		}
	}
	startPriorityEvalTime := time.Now()
	// 如果只有一个Node,则直接返回该node
	if len(filteredNodes) == 1 {
		return ScheduleResult{
			SuggestedHost:  filteredNodes[0].Name,
			EvaluatedNodes: 1 + len(failedPredicateMap),
			FeasibleNodes:  1,
		}, nil
	}

    //根据pod的信息，执行优选算法
	metaPrioritiesInterface := g.priorityMetaProducer(pod, g.nodeInfoSnapshot.NodeInfoMap)
    //在PrioritizeNodes中传入pod信息，和符合条件的node信息，进行优选
	priorityList, err := PrioritizeNodes(pod, g.nodeInfoSnapshot.NodeInfoMap, metaPrioritiesInterface, g.prioritizers, filteredNodes, g.extenders, g.framework, pluginContext)
	if err != nil {
		return result, err
	}
	trace.Step("Prioritizing done")

    //通过打分机制选择出最合适的node
	host, err := g.selectHost(priorityList)
	trace.Step("Selecting host done")
	return ScheduleResult{
		SuggestedHost:  host,
		EvaluatedNodes: len(filteredNodes) + len(failedPredicateMap),
		FeasibleNodes:  len(filteredNodes),
	}, err
}
```

最后，kube-scheduler在执行预选和优选过程中默认提供的调度算法，位置如下：
```go
// pkg/scheduler/algorithmprovider/defaults/defaults.go
func defaultPredicates() sets.String {
    return sets.NewString(
        predicates.NoVolumeZoneConflictPred,
        predicates.MaxEBSVolumeCountPred,
        predicates.MaxGCEPDVolumeCountPred,
        predicates.MaxAzureDiskVolumeCountPred,
        predicates.MaxCSIVolumeCountPred,
        predicates.MatchInterPodAffinityPred,
        predicates.NoDiskConflictPred,
        predicates.GeneralPred,
        predicates.CheckNodeMemoryPressurePred,
        predicates.CheckNodeDiskPressurePred,
        predicates.CheckNodePIDPressurePred,
        predicates.CheckNodeConditionPred,
        predicates.PodToleratesNodeTaintsPred,
        predicates.CheckVolumeBindingPred,
    )
}

func defaultPriorities() sets.String {
    return sets.NewString(
        priorities.SelectorSpreadPriority,
        priorities.InterPodAffinityPriority,
        priorities.LeastRequestedPriority,
        priorities.BalancedResourceAllocation,
        priorities.NodePreferAvoidPodsPriority,
        priorities.NodeAffinityPriority,
        priorities.TaintTolerationPriority,
        priorities.ImageLocalityPriority,
    )
}
```
这里我将主要的策略罗列并解释一下：

Predicates 预选策略是一组filter chain，所有node会经过file chain来层层过滤，最终筛选出合适的节点。

*一般策略*

**策略**           | **描述**     |   |
 -------------------|------------------------------------|-------------------------------------------------------
PodFitsResources                 | 用于判断当前node的资源是否满足pod的request的资源条件                               | 
PodFitsHost                      | 用于判断当前node的名字是否满足pod所指定的nodeName                             | 
PodFitsHostPorts                 | Pod对象拥有spec.hostPort属性时,用于判断当前node可用的端口是否满足pod所要求的端口占用   | 
PodMatchNodeSelector             | 用于判断当前node是否匹配pod所定义的nodeSelector或者nodeAffinity   | 

*Volume相关策略*

**策略**           | **描述**     |   |
 -------------------|------------------------------------|-------------------------------------------------------
NoDiskConflict                 | 用于判断多个pod所声明的volume是否有冲突，默认没有启用                              | 
MaxPDVolumeCountPredicate       | 用于判断某种volume是否已经超过所指定的数目                            | 
VolumeBindingPredicate	|用于检查pod所定义的volume的nodeAffinity是否与node的标签所匹配 | 
NoVolumeZoneConflict	|检查给定的zone限制前提下，检查如果在此主机上部署Pod是否存在卷冲突 | 
NoVolumeNodeConflict	|检查给定的Node限制前提下，检查如果在此主机上部署Pod是否存在卷冲突 | 
MaxEBSVolumeCount	|确保已挂载的EBS存储卷不超过设置的最大值，默认39 | 
MaxGCEPDVolumeCount	|确保已挂载的GCE存储卷不超过设置的最大值，默认16 | 
MaxAzureDiskVolumeCount	|确保已挂载的Azure存储卷不超过设置的最大值，默认16 | 
CheckVolumeBinding	|检查节点上已绑定和未绑定的PVC是否满足需求 | 

*Node相关策略*

**策略**           | **描述**     |   |
 -------------------|------------------------------------|-------------------------------------------------------
MatchNodeSelector	 | Pod对象拥有spec.nodeSelector属性时，检查Node节点的label定义是否满足Pod的NodeSelector属性需求| 
HostName	 | 如果Pod对象拥有spec.hostname属性，则检查节点名称是不是Pod指定的NodeName| 
PodToleratesNodeTaints	 | Pod对象拥有spec.tolerations属性时，仅关注NoSchedule和NoExecute两个效用标识的污点| 
PodToleratesNodeNoExecuteTaints	P | od对象拥有spec.tolerations属性时，，是否能接纳节点的NoExecute类型污点,默认没有启用| 
CheckNodeLabelPresence	 | 仅检查节点上指定的所有标签的存在性,默认没有启用| 
CheckServiceAffinity	 | 将相同Service的Pod对象放置在同一个或同一类节点上以提高效率,默认没有启用| 
NodeMemoryPressurePredicate	 | 检查当前node的内存是否充足，只有充足的时候才会调度到该node| 
CheckNodeMemoryPressure	 | 检查节点内存压力，如果压力过大，那就不会将pod调度至此| 
CheckNodeDiskPressure	 | 检查节点磁盘资源压力，如果压力过大，那就不会将pod调度至此| 
GeneralPredicates	 | 检查pod与主机上kubernetes相关组件是否匹配| 
CheckNodeCondition	 | 检查是否可以在节点报告磁盘、网络不可用或未准备好时将Pod调度其上| 

*Pod相关策略*

**策略**           | **描述**     |   |
 -------------------|------------------------------------|-------------------------------------------------------
 PodAffinityPredicate	|用于检查pod和该node上的pod是否和affinity以及anti-affinity规则匹配|
 MatchInterPodAffinity	|检查节点是否满足Pod对象亲和性或反亲和性条件|

当然对于上面的规则链调用是有一定顺序的，通常跟node相关的规则会先计算，这样就可以避免一些没有必要的规则校验，比如在一个内存严重不足的node上面计算pod的affinity是没有意义的。有一个问题这么多调用链，如果待选择的node节点很多会不会
导致过滤很慢，过滤函数启动多个goroutine并行计算来对node进行过滤，所以过滤速度可以得到保证。

经过预选策略得到一组待选node后进入优选策略

kubernetes用一组优先级函数处理每一个待选的主机。每一个优先级函数会返回一个0-10的分数，分数越高表示主机越“好”，同时每一个函数也会对应一个表示权重的值。最终主机的得分用以下公式计算得出：
```
finalScoreNode = (weight1 * priorityFunc1) + (weight2 * priorityFunc2) + … + (weightn * priorityFuncn)
```
Priorites策略也在随着版本演进而丰富，v1.0版本仅支持3个策略，v1.7支持10个策略，每项策略都有对应权重，最终根据权重计算节点总分。目前可用的Priorites策略有：

**策略**           | **描述**     |   |
 -------------------|------------------------------------|-------------------------------------------------------
SelectorSpreadPriority	|对于属于同一个service、replication controller的Pod，尽量分散在不同的主机上。如果指定了区域，则会尽量把Pod分散在不同区域的不同主机上。调度一个Pod的时候，先查找Pod对于的service或者replication controller，然后查找service或replication controller中已存在的Pod，主机上运行的已存在的Pod越少，主机的打分越高|
LeastRequestedPriority	|如果新的pod要分配给一个节点，这个节点的优先级就由节点空闲的那部分与总容量的比值（即（总容量-节点上pod的容量总和-新pod的容量）/总容量）来决定。CPU和memory权重相当，比值最大的节点的得分最高。需要注意的是，这个优先级函数起到了按照资源消耗来跨节点分配pods的作用。计算公式如下：cpu((capacity – sum(requested)) * 10 / capacity) + memory((capacity – sum(requested)) * 10 / capacity) / 2 |
BalancedResourceAllocation	|尽量选择在部署Pod后各项资源更均衡的机器。BalancedResourceAllocation不能单独使用，而且必须和LeastRequestedPriority同时使用，它分别计算主机上的cpu和memory的比重，主机的分值由cpu比重和memory比重的“距离”决定。计算公式如下：10 – abs(totalCpu/cpuNodeCapacity-totalMemory/memoryNodeCapacity)*10 |
NodeAffinityPriority	|节点亲和性选择策略。Node Selectors（调度时将pod限定在指定节点上），支持多种操作符（In, NotIn, Exists, DoesNotExist, Gt, Lt），而不限于对节点labels的精确匹配。另外，Kubernetes支持两种类型的选择器，一种是“hard（requiredDuringSchedulingIgnoredDuringExecution）”选择器，它保证所选的主机必须满足所有Pod对主机的规则要求。这种选择器更像是之前的nodeselector，在nodeselector的基础上增加了更合适的表现语法。另一种是“soft（preferresDuringSchedulingIgnoredDuringExecution）”选择器，它作为对调度器的提示，调度器会尽量但不保证满足NodeSelector的所有要求|
InterPodAffinityPriority	|pod亲和性选择策略,类似NodeAffinityPriority，提供两种选择器支持。有两个子策略podAffinity和podAntiAffinity |
NodePreferAvoidPodsPriority(权重1W)	|判断alpha.kubernetes.io/preferAvoidPods属性，设置权重为10000，覆盖其他策略 |
TaintTolerationPriority	|使用Pod中tolerationList与Node节点Taint进行匹配，配对成功的项越多，则得分越低 |
ImageLocalityPriority	|根据主机上是否已具备Pod运行的环境来打分，得分计算：不存在所需镜像，返回0分，存在镜像，镜像越大得分越高。默认没有启用 |
EqualPriority	|EqualPriority是一个优先级函数，它给予所有节点相等的权重（优先级） |
ServiceSpreadingPriority	|按Service和Replicaset归属计算Node上分布最少的同类Pod数量，得分计算：数量越少得分越高（作用于SelectorSpreadPriority相同，已经被SelectorSpreadPriority替换）|
MostRequestedPriority	|在ClusterAutoscalerProvider中，替换LeastRequestedPriority，给使用多资源的节点，更高的优先级。计算公式为：(cpu(10 * sum(requested) / capacity) + memory(10 * sum(requested) / capacity)) / 2 动态伸缩集群环境比较适用，会优先调度pod到使用率最高的主机节点，这样在伸缩集群时，就会腾出空闲机器，从而进行停机处理。默认没有启用 |

接下来会对scheduler 优选策略进行详细分析，请看 [Kube Scheduler 优选策略源码分析](https://wukaiying.github.io/k8s/kube-scheduler-priority-source-read/)

### 总结

整个调度流程图可以用下面这幅图描述

![scheduler调度流程图](https://pic.images.ac.cn/image/5ea033588dfb4.html)

1. 从待调度队列中取出一个pod，这块背后是由SchedulingQueue该模块实现，暂不进行深入分析

2. 如果pod被标记为待删除，则不进行调度

3. 加载调度策略，通过sched.schedule来对pod进行调度，如果调度成功会得到一个node节点名称（此时已经经过预选和优选完成）

3.1 预选优选流程

- pod做一些前置过滤
- 获取node数量 numNodes := g.cache.NodeTree().NumNodes()
- 执行预选算法，得到可用的node filteredNodes。filteredNodes = g.findNodesThatFit(pluginContext, pod)
- 如果预选完成没有合适的node,如果只有一个Node,则直接返回该node
- 在优选函数中传入pod信息，和符合条件的node信息，进行优选
- 通过打分机制选择出最合适的node，host= g.selectHost(priorityList)，完成scheduler调度
    
4. 如果调度失败，查看返回的错误中，该pod是否启动了抢占策略，如果没有则什么也不做，当pod再次被更新需要重新调度时，pod重新入队，重新调度

4.1 抢占流程
- pod做一些前置过滤
- 获取node数量 numNodes := g.cache.NodeTree().NumNodes()
- 执行预选算法，得到可用的node filteredNodes。filteredNodes = g.findNodesThatFit(pluginContext, pod)
- 如果预选完成没有合适的node,如果只有一个Node,则直接返回该node
- 在优选函数中传入pod信息，和符合条件的node信息，进行优选
- 通过打分机制选择出最合适的node，host= g.selectHost(priorityList)，完成scheduler调度
     
5. 执行绑定操作，也就是将pod.Spec.NodeName设置为选出来的node

6. 得到最合适的node,将pod.spec.nodename设置为该值，提交到api server写入etcd,下一个组件Kubelet watch api server，完成pod创建

具体方法调用链：

cmd/kube-scheduler/scheduler.go:32 NewSchedulerCommand()

                    ||
                    
cmd/kube-scheduler/app/server.go:116 runCommand() runCommand方法主要作用是校验scheduler 配置参数，生成scheduler配置文件，并通过Run方法来启动scheduler

                    ||
                    
cmd/kube-scheduler/app/server.go:165 Run()  启动scheduler

                    ||
                    
pkg/scheduler/scheduler.go:315 sched.scheduleOne()  执行预选，优选，抢占，得到最合适的node,将pod.spec.nodename设置为该值，提交到api server写入etcd


### 后续

很多公司结合自己业务，有需要扩展scheduler的需求，后续我会继续在一下两个方面研究：

1.scheduler 扩展实现自定义策略调度

2.scheduler GPU调度

扩展例子
example
[k8s-scheduler-extender-example](https://github.com/everpeace/k8s-scheduler-extender-example)

Gang Scheduling
[Kube-batch](https://github.com/kubernetes-sigs/kube-batch), gang scheduler 是某些领域，比如大数据、批量计算场景 常用的的调度方式，即讲一组资源当成一个 group，如果有 group 够用的资源就整个调度，或者整个不调度 (而传统的 kubernetes 的调度粒度为 pod). kubebatch 试图解决此类问题，并且想把这种通用的需求变成标准，解决所有类似的问题.

[gpushare-scheduler-extender](https://github.com/AliyunContainerService/gpushare-scheduler-extender)
为 gpu share divice 扩展的 scheduler，支持多个 pod 共享 gpu显存和 card. 目前的 device 机制能注册资源总量，但是对于调度来讲，信息不太够，因此 gpushare-scheduler-extender 提供了一层 filter 帮助判断 node 上是否有足够的 gpu 资源.




### 参考
https://kubernetes.io/zh/docs/reference/command-line-tools-reference/kube-scheduler/

https://blog.tianfeiyu.com/source-code-reading-notes/kubernetes/kube_scheduler_algorithm.html




















