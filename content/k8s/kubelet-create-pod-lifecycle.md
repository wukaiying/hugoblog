---
title: "Kubelet 创建pod流程分析"
date: 2020-04-28T10:38:28+08:00
draft: false
categories: ["k8s"]
keywords: ["k8s","kubelet"]  
description: "Kubelet 创建pod流程分析" 
summaryLength: 100
---

### 前言

上一节我们分析了kubelet启动的基本流程，这一节我们分析一下当有新的pod分配到当前节点时，kubelet如何创建一个pod

### syncLoop
上一节最后我们分析到`kl.syncLoop(updates, kl)`，是kubelet里面一个主循环逻辑，看一下它的具体实现：
它从不同的管道（文件、URL 和 apiserver）监听变化，并把它们汇聚起来。
当有新的变化发生时，它会调用对应的处理函数，保证 pod 处于期望的状态。
如果 pod 没有变化，它也会定期保证所有的容器和最新的期望状态保持一致。这个方法是 for 循环，不会退出。

```go
func (kl *Kubelet) syncLoop(updates <-chan kubetypes.PodUpdate, handler SyncHandler) {
	klog.Info("Starting kubelet main sync loop.")
	syncTicker := time.NewTicker(time.Second)
	defer syncTicker.Stop()
	housekeepingTicker := time.NewTicker(housekeepingPeriod)
	defer housekeepingTicker.Stop()
	plegCh := kl.pleg.Watch()
	const (
		base   = 100 * time.Millisecond
		max    = 5 * time.Second
		factor = 2
	)
	duration := base
	for {
		if err := kl.runtimeState.runtimeErrors(); err != nil {
			klog.Infof("skipping pod synchronization - %v", err)
			// exponential backoff
			time.Sleep(duration)
			duration = time.Duration(math.Min(float64(max), factor*float64(duration)))
			continue
		}
		// reset backoff if we have a success
		duration = base

		kl.syncLoopMonitor.Store(kl.clock.Now())
		if !kl.syncLoopIteration(updates, handler, syncTicker.C, housekeepingTicker.C, plegCh) {
			break
		}
		kl.syncLoopMonitor.Store(kl.clock.Now())
	}
}
```

我们看一下for循环中的核心逻辑`syncLoopIteration`

```go
func (kl *Kubelet) syncLoopIteration(configCh <-chan kubetypes.PodUpdate, handler SyncHandler,
	syncCh <-chan time.Time, housekeepingCh <-chan time.Time, plegCh <-chan *pleg.PodLifecycleEvent) bool {
	select {
	case u, open := <-configCh:
		// Update from a config source; dispatch it to the right handler
		// callback.
		if !open {
			klog.Errorf("Update channel is closed. Exiting the sync loop.")
			return false
		}

		switch u.Op {
		case kubetypes.ADD:
			klog.V(2).Infof("SyncLoop (ADD, %q): %q", u.Source, format.Pods(u.Pods))
			// After restarting, kubelet will get all existing pods through
			// ADD as if they are new pods. These pods will then go through the
			// admission process and *may* be rejected. This can be resolved
			// once we have checkpointing.
			handler.HandlePodAdditions(u.Pods)
		case kubetypes.UPDATE:
			klog.V(2).Infof("SyncLoop (UPDATE, %q): %q", u.Source, format.PodsWithDeletionTimestamps(u.Pods))
			handler.HandlePodUpdates(u.Pods)
		case kubetypes.REMOVE:
			klog.V(2).Infof("SyncLoop (REMOVE, %q): %q", u.Source, format.Pods(u.Pods))
			handler.HandlePodRemoves(u.Pods)
		case kubetypes.RECONCILE:
			klog.V(4).Infof("SyncLoop (RECONCILE, %q): %q", u.Source, format.Pods(u.Pods))
			handler.HandlePodReconcile(u.Pods)
		case kubetypes.DELETE:
			klog.V(2).Infof("SyncLoop (DELETE, %q): %q", u.Source, format.Pods(u.Pods))
			// DELETE is treated as a UPDATE because of graceful deletion.
			handler.HandlePodUpdates(u.Pods)
		case kubetypes.RESTORE:
			klog.V(2).Infof("SyncLoop (RESTORE, %q): %q", u.Source, format.Pods(u.Pods))
			// These are pods restored from the checkpoint. Treat them as new
			// pods.
			handler.HandlePodAdditions(u.Pods)
		case kubetypes.SET:
			// TODO: Do we want to support this?
			klog.Errorf("Kubelet does not support snapshot update")
		}

		if u.Op != kubetypes.RESTORE {
			// If the update type is RESTORE, it means that the update is from
			// the pod checkpoints and may be incomplete. Do not mark the
			// source as ready.

			// Mark the source ready after receiving at least one update from the
			// source. Once all the sources are marked ready, various cleanup
			// routines will start reclaiming resources. It is important that this
			// takes place only after kubelet calls the update handler to process
			// the update to ensure the internal pod cache is up-to-date.
			kl.sourcesReady.AddSource(u.Source)
		}
	case e := <-plegCh:
		if isSyncPodWorthy(e) {
			// PLEG event for a pod; sync it.
			if pod, ok := kl.podManager.GetPodByUID(e.ID); ok {
				klog.V(2).Infof("SyncLoop (PLEG): %q, event: %#v", format.Pod(pod), e)
				handler.HandlePodSyncs([]*v1.Pod{pod})
			} else {
				// If the pod no longer exists, ignore the event.
				klog.V(4).Infof("SyncLoop (PLEG): ignore irrelevant event: %#v", e)
			}
		}

		if e.Type == pleg.ContainerDied {
			if containerID, ok := e.Data.(string); ok {
				kl.cleanUpContainersInPod(e.ID, containerID)
			}
		}
	case <-syncCh:
		// Sync pods waiting for sync
		podsToSync := kl.getPodsToSync()
		if len(podsToSync) == 0 {
			break
		}
		klog.V(4).Infof("SyncLoop (SYNC): %d pods; %s", len(podsToSync), format.Pods(podsToSync))
		handler.HandlePodSyncs(podsToSync)
	case update := <-kl.livenessManager.Updates():
		if update.Result == proberesults.Failure {
			// The liveness manager detected a failure; sync the pod.

			// We should not use the pod from livenessManager, because it is never updated after
			// initialization.
			pod, ok := kl.podManager.GetPodByUID(update.PodUID)
			if !ok {
				// If the pod no longer exists, ignore the update.
				klog.V(4).Infof("SyncLoop (container unhealthy): ignore irrelevant update: %#v", update)
				break
			}
			klog.V(1).Infof("SyncLoop (container unhealthy): %q", format.Pod(pod))
			handler.HandlePodSyncs([]*v1.Pod{pod})
		}
	case <-housekeepingCh:
		if !kl.sourcesReady.AllReady() {
			// If the sources aren't ready or volume manager has not yet synced the states,
			// skip housekeeping, as we may accidentally delete pods from unready sources.
			klog.V(4).Infof("SyncLoop (housekeeping, skipped): sources aren't ready yet.")
		} else {
			klog.V(4).Infof("SyncLoop (housekeeping)")
			if err := handler.HandlePodCleanups(); err != nil {
				klog.Errorf("Failed cleaning pods: %v", err)
			}
		}
	}
	return true
}
```
可以看到它从如下管道中获取信息：

configCh：读取配置事件的管道，就是之前讲过的通过文件、URL 或 apiserver 得到对pod资源做出修改的事件

syncCh：定时器管道，每次隔一段事件去同步最新保存的 pod 状态

houseKeepingCh：housekeeping 事件的管道，做 pod 清理工作

plegCh：PLEG 状态，如果 pod 的状态发生改变（因为某些情况被杀死，被暂停等），kubelet 也要做处理

livenessManager.Updates()：健康检查，发现某个 pod 不可用，一般也要对它进行重启

需要注意的是， switch-case 语句从管道中读取数据的时候，不像一般情况下那样会从上到下按照顺序，只要任何管道中有数据，switch 就会选择执行对应的 case 语句。

每个管道的处理思路大同小异，我们只分析用户通过 apiserver 添加新 pod 的情况，也就是 `handler.HandlePodAdditions(u.Pods)` 这句话的处理逻辑。

### HandlePodAdditions

入参是一个pod结构体数组，对每一个pod执行如下操作：

- 1.先把所有Pod按照时间顺序进行排序
- 2.将所有pod加入到podManager中，对于不在podManager中的pod,如果podManager中找不到某个pod，则认为该pod被删除了
- 3.判断是不是静态pod，如果是则进入静态pod 创建逻辑
- 4.如果是普通pod,首先会检查该pod能否在当前节点上创建
- 5.如果可以创建则使用`dispatchWork`将创建pod的任务异步下发给`podWorkers`
- 6.将pod加入到`probeManager`，启动readiness和liveness讲课检查

```go
//pkg/kubelet/kubelet.go:2040
func (kl *Kubelet) HandlePodAdditions(pods []*v1.Pod) {
	start := kl.clock.Now()
    //把所有Pod按照时间顺序进行排序
	sort.Sort(sliceutils.PodsByCreationTime(pods))
	
    //将所有pod加入到podManager中，对于不在podManager中的pod,如果podManager中找不到某个pod，则认为该pod被删除了
	for _, pod := range pods {
		existingPods := kl.podManager.GetPods()
		// Always add the pod to the pod manager. Kubelet relies on the pod
		// manager as the source of truth for the desired state. If a pod does
		// not exist in the pod manager, it means that it has been deleted in
		// the apiserver and no action (other than cleanup) is required.
		kl.podManager.AddPod(pod)

        //判断加入的pod是不是mirror pod，也就是static pod，俗称静态pod,
        //静态pod无法通过master来创建和删除，不与任何一种k8s controller资源关联，它只能通过kubelet来创建，
        //通过设置kubelet 中--pod-manifest-path参数来指定pod yaml文件的路径，kubelet定时扫描创建pod
        //静态pod的主要作用就是他不会被oom回收，系统组件可以使用静态pod的方式部署
		if kubepod.IsMirrorPod(pod) {
			kl.handleMirrorPod(pod, start)
			continue
		}

		if !kl.podIsTerminated(pod) {
		
			activePods := kl.filterOutTerminatedPods(existingPods)

			//校验该pod能否在当前节点上创建
			if ok, reason, message := kl.canAdmitPod(activePods, pod); !ok {
				kl.rejectPod(pod, reason, message)
				continue
			}
		}
		mirrorPod, _ := kl.podManager.GetMirrorPodByPod(pod)
    
        //任务分发，资源为pod，对资源做哪种操作kubetypes.SyncPodCreate
		kl.dispatchWork(pod, kubetypes.SyncPodCreate, mirrorPod, start)
        //将pod加入probeManager，启动readiness和liveness健康检查
		kl.probeManager.AddPod(pod)
	}
}
```

### dispatchWork

现在我们来分析当创建pod的任务下发后，`dispatchWork`里面做了什么

```go
func (kl *Kubelet) dispatchWork(pod *v1.Pod, syncType kubetypes.SyncPodType, mirrorPod *v1.Pod, start time.Time) {
	//如果pod被标记为删除，则不做处理
    if kl.podIsTerminated(pod) {
		if pod.DeletionTimestamp != nil {
			kl.statusManager.TerminatePod(pod)
		}
		return
	}
	
    //填充UpdatePodOptions，然后调用UpdatePod方法
	kl.podWorkers.UpdatePod(&UpdatePodOptions{
		Pod:        pod,
		MirrorPod:  mirrorPod,
		UpdateType: syncType,
		OnCompleteFunc: func(err error) {
			if err != nil {
				metrics.PodWorkerDuration.WithLabelValues(syncType.String()).Observe(metrics.SinceInSeconds(start))
				metrics.DeprecatedPodWorkerLatency.WithLabelValues(syncType.String()).Observe(metrics.SinceInMicroseconds(start))
			}
		},
	})
	// Note the number of containers for new pods.
	if syncType == kubetypes.SyncPodCreate {
		metrics.ContainersPerPodCount.Observe(float64(len(pod.Spec.Containers)))
	}
}
```

进入`kl.podWorkers.UpdatePod`方法查看，我们发现UpdatePod中主要包含一个方法`managePodLoop`，在进入
`managePodLoop`中我们发现`managePodLoop` 调用 `syncPodFn` 方法去同步 pod，syncPodFn 实际上就是 `kubelet.SyncPod`，
我们直接进入`kubelet.SyncPod`中看创建pod的具体流程：

- 1. 对syncPod中传入参数进行判断，如果是删除Pod,则直接删除
- 2. 同步 podStatus 到 kubelet.statusManager
- 3. 判断网络插件是否ready，如果没有ready，则只能以hostnetwork的方式启动pod
- 4. 检测是否为pod启用了cgroups-per-qos，如果是则更新pod的cgroup 
- 5. 为pod创建数据目录
- 6. 将pod中声明的volume进行挂载
- 7. 获取pod依赖的所有secret
- 8. 调用containerruntime 开始创建容器


```go
pkg/kubelet/kubelet.go:1481
func (kl *Kubelet) syncPod(o syncPodOptions) error {
	// pull out the required options
	pod := o.pod
	mirrorPod := o.mirrorPod
	podStatus := o.podStatus
	updateType := o.updateType

	//对pod的操作是删除pod，则删除pod
	if updateType == kubetypes.SyncPodKill {
		if err := kl.killPod(pod, nil, podStatus, killPodOptions.PodTerminationGracePeriodSecondsOverride); err != nil {
			utilruntime.HandleError(err)
			return err
		}
		return nil
	}
	
    //对pod的操作是创建pod，记录kubelet第一次发现该pod的时间
	if updateType == kubetypes.SyncPodCreate {
		if !firstSeenTime.IsZero() {
			// This is the first time we are syncing the pod. Record the latency
			// since kubelet first saw the pod if firstSeenTime is set.
			metrics.PodWorkerStartDuration.Observe(metrics.SinceInSeconds(firstSeenTime))
			metrics.DeprecatedPodWorkerStartLatency.Observe(metrics.SinceInMicroseconds(firstSeenTime))
		} else {
			klog.V(3).Infof("First seen time not recorded for pod %q", pod.UID)
		}
	}

    //将podStatus同步到kubelet status manager
    apiPodStatus := kl.generateAPIPodStatus(pod, podStatus)
	

	// 判断网络插件是否ready，如果没有ready，则只能以hostnetwork的方式启动pod
	if err := kl.runtimeState.networkErrors(); err != nil && !kubecontainer.IsHostNetworkPod(pod) {
		kl.recorder.Eventf(pod, v1.EventTypeWarning, events.NetworkNotReady, "%s: %v", NetworkNotReadyErrorMsg, err)
		return fmt.Errorf("%s: %v", NetworkNotReadyErrorMsg, err)
	}

	//为pod设置cgroup 如果pod启用了cgroups-per-qos
	pcm := kl.containerManager.NewPodContainerManager()
    if err := kl.containerManager.UpdateQOSCgroups(); err != nil {
        klog.V(2).Infof("Failed to update QoS cgroups while syncing pod: %v", err)
    }

	// 为pod创建数据目录
	if err := kl.makePodDataDirs(pod); err != nil {
		kl.recorder.Eventf(pod, v1.EventTypeWarning, events.FailedToMakePodDataDirectories, "error making pod data directories: %v", err)
		klog.Errorf("Unable to make pod data directories for pod %q: %v", format.Pod(pod), err)
		return err
	}

	// 将pod中声明的volume进行挂载
	if !kl.podIsTerminated(pod) {
		// Wait for volumes to attach/mount
		if err := kl.volumeManager.WaitForAttachAndMount(pod); err != nil {
			kl.recorder.Eventf(pod, v1.EventTypeWarning, events.FailedMountVolume, "Unable to attach or mount volumes: %v", err)
			klog.Errorf("Unable to attach or mount volumes for pod %q: %v; skipping pod", format.Pod(pod), err)
			return err
		}
	}

	// 获取pod依赖的所有secret
	pullSecrets := kl.getPullSecretsForPod(pod)

	// 调用containerruntime 开始创建容器
	result := kl.containerRuntime.SyncPod(pod, podStatus, pullSecrets, kl.backOff)
	kl.reasonCache.Update(pod.UID, result)
	if err := result.Error(); err != nil {
		// Do not return error if the only failures were pods in backoff
		for _, r := range result.SyncResults {
			if r.Error != kubecontainer.ErrCrashLoopBackOff && r.Error != images.ErrImagePullBackOff {
				// Do not record an event here, as we keep all event logging for sync pod failures
				// local to container runtime so we get better errors
				return err
			}
		}

		return nil
	}

	return nil
}
```



### 创建容器逻辑

上面一小节创建了pod所必须的所有内容，但是最核心还是pod中的容器的创建，我们来分析一下，进入`kl.containerRuntime.SyncPod`,
核心逻辑如下：
- 1. 创建 PodSandbox 容器
- 2. 启动 initContainer 容器
- 3. 启动业务容器

```go
//pkg/kubelet/kuberuntime/kuberuntime_manager.go:631
func (m *kubeGenericRuntimeManager) SyncPod(pod *v1.Pod, _ v1.PodStatus, podStatus *kubecontainer.PodStatus, pullSecrets []v1.Secret, backOff *flowcontrol.Backoff) (result kubecontainer.PodSyncResult) {
	...
	// 1. 创建 PodSandbox 容器
	podSandboxID, msg, err = m.createPodSandbox(pod, podContainerChanges.Attempt)
	...
	
	// 2. 启动 initContainer 容器
	if msg, err := m.startContainer(podSandboxID, podSandboxConfig, container, pod, podStatus, pullSecrets, podIP); err != nil {
	...		
	// 3. 启动业务容器
	for _, idx := range podContainerChanges.ContainersToStart {
	...
	if msg, err := m.startContainer(podSandboxID, podSandboxConfig, container, pod, podStatus, pullSecrets, podIP); err != nil {
	...
}
```
至此一个pod被完整的启动起来


### 总结

主要总结下kubelet创建pod的核心流程：

- 1. 对syncPod(kubelet核心循环逻辑)中传入参数进行判断，如果是删除Pod,则直接删除
- 2. 同步 podStatus 到 kubelet.statusManager
- 3. 判断网络插件是否ready，如果没有ready，则只能以hostnetwork的方式启动pod
- 4. 检测是否为pod启用了cgroups-per-qos，如果是则更新pod的cgroup 
- 5. 为pod创建数据目录
- 6. 将pod中声明的volume进行挂载
- 7. 获取pod依赖的所有secret
- 8. 调用containerruntime 开始创建容器
    - 8.1. 创建 PodSandbox 容器
    - 8.2. 启动 initContainer 容器
    - 8.3. 启动业务容器

### 参考
https://cizixs.com/2017/06/07/kubelet-source-code-analysis-part-2/
