---
title: "Kube Scheduler 优选策略源码分析"
date: 2020-04-24T15:32:13+08:00
draft: false
categories: ["k8s"]
keywords: ["k8s","scheduler","priority"]  
description: "k8s scheduler 优选策略源码分析" 
summaryLength: 100
---

### 版本环境
- kubernetes版本：kubernetes:v1.16.0

- go环境：go version go1.13.4 darwin/amd64

### 前言

预选阶段完成后，进入优选阶段，将需要调度的Pod列表和Node列表传入各种优选算法进行打分，最终整合成结果集HostPriorityList

```go
//pkg/scheduler/core/generic_scheduler.go:189
// Schedule tries to schedule the given pod to one of the nodes in the node list.
// If it succeeds, it will return the name of the node.
// If it fails, it will return a FitError error with reasons.
func (g *genericScheduler) Schedule(pod *v1.Pod, pluginContext *framework.PluginContext) (result ScheduleResult, err error) {
	
    ...

	trace.Step("Basic checks done")
	startPredicateEvalTime := time.Now()
	filteredNodes, failedPredicateMap, filteredNodesStatuses, err := g.findNodesThatFit(pluginContext, pod)
	if err != nil {
		return result, err
	}

	// Run "postfilter" plugins.
	postfilterStatus := g.framework.RunPostFilterPlugins(pluginContext, pod, filteredNodes, filteredNodesStatuses)
	if !postfilterStatus.IsSuccess() {
		return result, postfilterStatus.AsError()
	}

	if len(filteredNodes) == 0 {
		return result, &FitError{
			Pod:                   pod,
			NumAllNodes:           numNodes,
			FailedPredicates:      failedPredicateMap,
			FilteredNodesStatuses: filteredNodesStatuses,
		}
	}
	trace.Step("Computing predicates done")

    //上面已经完成初步过滤和预选，现在进入优选
	metaPrioritiesInterface := g.priorityMetaProducer(pod, g.nodeInfoSnapshot.NodeInfoMap)
	priorityList, err := PrioritizeNodes(pod, g.nodeInfoSnapshot.NodeInfoMap, metaPrioritiesInterface, g.prioritizers, filteredNodes, g.extenders, g.framework, pluginContext)
	if err != nil {
		return result, err
	}
	trace.Step("Prioritizing done")

	host, err := g.selectHost(priorityList)
	trace.Step("Selecting host done")
	return ScheduleResult{
		SuggestedHost:  host,
		EvaluatedNodes: len(filteredNodes) + len(failedPredicateMap),
		FeasibleNodes:  len(filteredNodes),
	}, err
}
```

接下来我们看一下优选算法的具体实现，进入`priorityList, err := PrioritizeNodes(...)` 查看

```go
// PrioritizeNodes prioritizes the nodes by running the individual priority functions in parallel.
// Each priority function is expected to set a score of 0-10
// 0 is the lowest priority score (least preferred node) and 10 is the highest
// Each priority function can also have its own weight
// The node scores returned by the priority function are multiplied by the weights to get weighted scores
// All scores are finally combined (added) to get the total weighted scores of all nodes
func PrioritizeNodes(
	pod *v1.Pod,
	nodeNameToInfo map[string]*schedulernodeinfo.NodeInfo,
	meta interface{},
	priorityConfigs []priorities.PriorityConfig,
	nodes []*v1.Node,
	extenders []algorithm.SchedulerExtender,
	framework framework.Framework,
	pluginContext *framework.PluginContext) (schedulerapi.HostPriorityList, error) {
...
```

根据上面注释描述我们可以总结出优选算法的基本流程：
- 每一个优选函数之间是相互独立的，且每个优选函数都有一个权重值
- 每个优选函数可以并行执行，他为每个node进行打分，分数为[0-10]
- 然后每个node的得分计算公式为每个优选函数分数乘以该优选函数对应的权重，并累计相加，为该node最后得分


看一下`HostPriorityList`，这个HostPriorityList数组保存了每个Node的名字和它对应的分数，最后选出分数最高的Node对Pod进行绑定和调度

```go
type HostPriority struct {
	Host string
	Score int
}
type HostPriorityList []HostPriority
```

假设有三个node(N1,N2,N3)进入优选，有三个优选算法(A1,A2,A3)，计算结果如下：

**N1**           | **N2**     | **N3**  | |
 -------------------|------------------------------------|--------------------------------|-----------------------	
A1	|{ Name:“node1”,Score:5,PriorityConfig:{…weight:1}}	|{ Name:“node2”,Score:3,PriorityConfig:{…weight:1}}	|{ Name:“node3”,Score:1,PriorityConfig:{…weight:1}}|
A2	|{ Name:“node1”,Score:6,PriorityConfig:{…weight:1}}	|{ Name:“node2”,Score:2,PriorityConfig:{…weight:1}}	|{ Name:“node3”,Score:3,PriorityConfig:{…weight:1}}|
A3	|{ Name:“node1”,Score:4,PriorityConfig:{…weight:1}}	|{ Name:“node2”,Score:7,PriorityConfig:{…weight:1.}}	|{ Name:“node3”,Score:2,PriorityConfig:{…weight:1}}|

每个node最后得分如下：
```
HostPriorityList =[{ Name:"node1",Score:15},{ Name:"node2",Score:12},{ Name:"node3",Score:6}]
```

### 打分算法分析

```go
//pkg/scheduler/core/generic_scheduler.go:696
func PrioritizeNodes(
	pod *v1.Pod,
	nodeNameToInfo map[string]*schedulernodeinfo.NodeInfo,
	meta interface{},
	priorityConfigs []priorities.PriorityConfig,
	nodes []*v1.Node,
	extenders []algorithm.SchedulerExtender,
	framework framework.Framework,
	pluginContext *framework.PluginContext) (schedulerapi.HostPriorityList, error) {
	
    //如果没有添加优选策略，则每个node的得分都是1
	if len(priorityConfigs) == 0 && len(extenders) == 0 {
		result := make(schedulerapi.HostPriorityList, 0, len(nodes))
		for i := range nodes {
			hostPriority, err := EqualPriorityMap(pod, meta, nodeNameToInfo[nodes[i].Name])
			if err != nil {
				return nil, err
			}
			result = append(result, hostPriority)
		}
		return result, nil
	}    
    ...
	results := make([]schedulerapi.HostPriorityList, len(priorityConfigs), len(priorityConfigs))

	// DEPRECATED: we can remove this when all priorityConfigs implement the
	// Map-Reduce pattern.
    //遍历每一个priorityConfigs，判断每一个priorityConfigs中是否包含Function，如果有则使用Function方式给所有nodes计算分数，一次遍历后，nodes中的每个节点得到了优选算法1的打分
    //整个遍历完成后，每个node都会有n个优选算法的打分
    //该方式目前已经不再推荐，后续都会改成map-reduce方式
	for i := range priorityConfigs {
		if priorityConfigs[i].Function != nil {
			wg.Add(1)
			go func(index int) {
				defer wg.Done()
				var err error
                //每个优选算法启动一个goroutine来计算该node在该算法下的分数[0-10]
				results[index], err = priorityConfigs[index].Function(pod, nodeNameToInfo, nodes)
				if err != nil {
					appendError(err)
				}
			}(i)
		} else {
			results[i] = make(schedulerapi.HostPriorityList, len(nodes))
		}
	}


    //使用map-reduce方式来计算分数
	workqueue.ParallelizeUntil(context.TODO(), 16, len(nodes), func(index int) {
		nodeInfo := nodeNameToInfo[nodes[index].Name]
		for i := range priorityConfigs {
            //跳过function 打分类型
			if priorityConfigs[i].Function != nil {
				continue
			}

			var err error
            //通过map函数来进行并行计算,index表示第几个优选函数
			results[i][index], err = priorityConfigs[i].Map(pod, meta, nodeInfo)
			if err != nil {
				appendError(err)
				results[i][index].Host = nodes[index].Name
			}
		}
	})

	for i := range priorityConfigs {
		if priorityConfigs[i].Reduce == nil {
			continue
		}
		wg.Add(1)
		go func(index int) {
			defer wg.Done()
            //使用reduce函数来进行分数聚合
			if err := priorityConfigs[index].Reduce(pod, meta, nodeNameToInfo, results[index]); err != nil {
				appendError(err)
			}
			if klog.V(10) {
				for _, hostPriority := range results[index] {
					klog.Infof("%v -> %v: %v, Score: (%d)", util.GetPodFullName(pod), hostPriority.Host, priorityConfigs[index].Name, hostPriority.Score)
				}
			}
		}(i)
	}
	// Wait for all computations to be finished.
	wg.Wait()
	if len(errs) != 0 {
		return schedulerapi.HostPriorityList{}, errors.NewAggregate(errs)
	}

	//计算每个node的最终得分，计算方式为每个node中所有优选算法分数乘以权重之和
	for i := range nodes {
		result = append(result, schedulerapi.HostPriority{Host: nodes[i].Name, Score: 0})
		for j := range priorityConfigs {
			result[i].Score += results[j][i].Score * priorityConfigs[j].Weight
		}

		for j := range scoresMap {
			result[i].Score += scoresMap[j][i].Score
		}
	}

	if klog.V(10) {
		for i := range result {
			klog.Infof("Host %s => Score %d", result[i].Host, result[i].Score)
		}
	}
	return result, nil
}
```

### Function和Map-Reduce实例分析

从上面的分析我们可以看到，优选过程计算分数有两种方式，分别是Function类型，和Map-Reduce类型，这里我们各自选取一个例子来做分析：

*InterPodAffinityPriority(Function类型)*

```go
//pkg/scheduler/algorithm/priorities/interpod_affinity.go：99

func (ipa *InterPodAffinity) CalculateInterPodAffinityPriority(pod *v1.Pod, nodeNameToInfo map[string]*schedulernodeinfo.NodeInfo, nodes []*v1.Node) (schedulerapi.HostPriorityList, error) {
	affinity := pod.Spec.Affinity
	hasAffinityConstraints := affinity != nil && affinity.PodAffinity != nil
	hasAntiAffinityConstraints := affinity != nil && affinity.PodAntiAffinity != nil
    ...
	//上面省略的步骤，是已经为每一个node，计算出了一个初始分数，该分数可能大于10，下面会通过算法，会将每一个node的处理在10分以内

    //遍历每一个node的分数，选出一个最高得分，选出一个最低得分
	for i := range nodes {
		if pm.counts[i] > maxCount {
			maxCount = pm.counts[i]
		}
		if pm.counts[i] < minCount {
			minCount = pm.counts[i]
		}
	}

	// calculate final priority score for each node
	result := make(schedulerapi.HostPriorityList, 0, len(nodes))
    //计算最高分和最低分的差值
	maxMinDiff := maxCount - minCount
	for i, node := range nodes {
		fScore := float64(0)
        //如果差值大于0
		if maxMinDiff > 0 {
            //MaxPriority=10，假设当前node的计算结果是5，最大count是20，最小count是-3，那么这里就是10*[5-(-3)/20-(-3)]
            //这里fscore肯定小10
			fScore = float64(schedulerapi.MaxPriority) * (float64(pm.counts[i]-minCount) / float64(maxCount-minCount))
		}
        // 如果分差不大于0，这时候int(fScore)也就是0，对于各个node的结果都是0；
		result = append(result, schedulerapi.HostPriority{Host: node.Name, Score: int(fScore)})
		if klog.V(10) {
			klog.Infof("%v -> %v: InterPodAffinityPriority, Score: (%d)", pod.Name, node.Name, int(fScore))
		}
	}
	return result, nil
}
```
我们可以发现最终这个函数计算出了每个node的分值，这个分值在[0-10]之间。所以说到底Function做的事情就是根据一定的规则给每个node赋一个分值，这个分值要求在[0-10]之间，然后把这个HostPriorityList返回就行。

*CalculateNodeAffinityPriorityMap(Map方式)*

```go
//pkg/scheduler/algorithm/priorities/node_affinity.go:34

// CalculateNodeAffinityPriorityMap prioritizes nodes according to node affinity scheduling preferences
// indicated in PreferredDuringSchedulingIgnoredDuringExecution. Each time a node matches a preferredSchedulingTerm,
// it will get an add of preferredSchedulingTerm.Weight. Thus, the more preferredSchedulingTerms
// the node satisfies and the more the preferredSchedulingTerm that is satisfied weights, the higher
// score the node gets.

func CalculateNodeAffinityPriorityMap(pod *v1.Pod, meta interface{}, nodeInfo *schedulernodeinfo.NodeInfo) (schedulerapi.HostPriority, error) {
	node := nodeInfo.Node()
	if node == nil {
		return schedulerapi.HostPriority{}, fmt.Errorf("node not found")
	}

	// default is the podspec.
	affinity := pod.Spec.Affinity
	if priorityMeta, ok := meta.(*priorityMetadata); ok {
		// We were able to parse metadata, use affinity from there.
		affinity = priorityMeta.affinity
	}

	var count int32
	// A nil element of PreferredDuringSchedulingIgnoredDuringExecution matches no objects.
	// An element of PreferredDuringSchedulingIgnoredDuringExecution that refers to an
	// empty PreferredSchedulingTerm matches all objects.
	if affinity != nil && affinity.NodeAffinity != nil && affinity.NodeAffinity.PreferredDuringSchedulingIgnoredDuringExecution != nil {
		// Match PreferredDuringSchedulingIgnoredDuringExecution term by term.
		for i := range affinity.NodeAffinity.PreferredDuringSchedulingIgnoredDuringExecution {
			preferredSchedulingTerm := &affinity.NodeAffinity.PreferredDuringSchedulingIgnoredDuringExecution[i]
			if preferredSchedulingTerm.Weight == 0 {
				continue
			}

			// TODO: Avoid computing it for all nodes if this becomes a performance problem.
			nodeSelector, err := v1helper.NodeSelectorRequirementsAsSelector(preferredSchedulingTerm.Preference.MatchExpressions)
			if err != nil {
				return schedulerapi.HostPriority{}, err
			}
			if nodeSelector.Matches(labels.Set(node.Labels)) {
				count += preferredSchedulingTerm.Weight
			}
		}
	}

	return schedulerapi.HostPriority{
		Host:  node.Name,
		Score: int(count),
	}, nil
}

// CalculateNodeAffinityPriorityReduce is a reduce function for node affinity priority calculation.
var CalculateNodeAffinityPriorityReduce = NormalizeReduce(schedulerapi.MaxPriority, false)
```
撇开具体的亲和性计算细节，我们可以发现这个的count没有特定的规则，可能会加到10以上；另外这里的返回值是HostPriority类型，前面的Function返回了HostPriorityList类型。
我们来看下他对应的reduce函数

```go
//pkg/scheduler/algorithm/priorities/reduce.go:28

// NormalizeReduce generates a PriorityReduceFunction that can normalize the result
// scores to [0, maxPriority]. If reverse is set to true, it reverses the scores by
// subtracting it from maxPriority.
func NormalizeReduce(maxPriority int, reverse bool) PriorityReduceFunction {
	return func(
		_ *v1.Pod,
		_ interface{},
		_ map[string]*schedulernodeinfo.NodeInfo,
		result schedulerapi.HostPriorityList) error {

		var maxCount int
        //选出最大的分数
		for i := range result {
			if result[i].Score > maxCount {
				maxCount = result[i].Score
			}
		}

        ///最大分数为0,所有node得分为0
		if maxCount == 0 {
			if reverse {
				for i := range result {
					result[i].Score = maxPriority
				}
			}
			return nil
		}

		for i := range result {
			score := result[i].Score
            
            //举个例子：10*(5/20)
			score = maxPriority * score / maxCount
			if reverse {
                // 如果score是3，得到7；如果score是4，得到6，结果反转；
				score = maxPriority - score
			}

			result[i].Score = score
		}
		return nil
	}
}
```

最终通过reduce也可以得到每个node的分数，范围也是[0-10]

### 总结

Function：一个算法一次性计算出所有node的Score，这个Score的范围是规定的[0-10]

Map-Reduce：一个Map算法计算1个node的Score，这个Score可以灵活处理，可能是20，可能是-3；Map过程并发进行；最终得到的结果result通过Reduce归约，将这个算法对应的所有node的分值归约为[0-10]
