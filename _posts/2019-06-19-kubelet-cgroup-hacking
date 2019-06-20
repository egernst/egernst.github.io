---
layout: post
title: kubelet cgroup hacking notes
category: posts
---

# Cgroup management with respect to pod creation

Kubelet manages pod cgroups, though in a different way depending on the QoS of the pod. `guaranteed` cgroups are placed at the top of the `kubepods` hierarchy, while `best-effort`, `burstable` pods, while `guaranteed` are placed within separate best-effort and burstable cgroups under kubepods.

Pod management is carried out in Kubelet's [podSync function](https://github.com/kubernetes/kubernetes/blob/c47ab33e670cd5589b07418457e39b25cea6512f/pkg/kubelet/kubelet.go#L1446-L1690). When a pod is created, the QoS cgroups are updated (if burstable) and then the new pod cgroup is placed in the correct parent cgroup, depending on the QoS.

## QoS Cgroups

If creating a pod, then the QoS cgroups may need to be updated.  This is done [here](https://github.com/kubernetes/kubernetes/blob/c47ab33e670cd5589b07418457e39b25cea6512f/pkg/kubelet/kubelet.go#L1613):
```go
1613     if err := kl.containerManager.UpdateQOSCgroups()
```

This is handled by containerManagerImpl, which implements the container manager interface. [See it here](https://github.com/kubernetes/kubernetes/blob/c47ab33e670cd5589b07418457e39b25cea6512f/pkg/kubelet/cm/container_manager_linux.go#L538-L540)
```
538 func (cm *containerManagerImpl) UpdateQOSCgroups() error {
539         return cm.qosContainerManager.UpdateCgroups()
540 }
```

Because everything is an interface, qosContainerManagerImpl implements the qosContainerManager [here](https://github.com/kubernetes/kubernetes/blob/c47ab33e670cd5589b07418457e39b25cea6512f/pkg/kubelet/cm/qos_container_manager_linux.go#L270)

This is where the QoS cgroup fun happens in Kubelet.

### CPU Handling

CPU cgroups are handled created [here](https://github.com/kubernetes/kubernetes/blob/c47ab33e670cd5589b07418457e39b25cea6512f/pkg/kubelet/cm/qos_container_manager_linux.go#L285-L288):

As seen below, the QoS cgroup for Burstable will be updated if CPU requests are specified for the pod(s). Since, by definition best-effort does not define requests/limits, default value of 2 is (always) applied: 
```go
169 func (m *qosContainerManagerImpl) setCPUCgroupConfig(configs map[v1.PodQOSClass]*CgroupConfig) error {
170         pods := m.activePods()
171         burstablePodCPURequest := int64(0)
172         for i := range pods {
173                 pod := pods[i]
174                 qosClass := v1qos.GetPodQOS(pod)
175                 if qosClass != v1.PodQOSBurstable {
176                         // we only care about the burstable qos tier
177                         continue
178                 }
179                 req, _ := resource.PodRequestsAndLimits(pod)
180                 if request, found := req[v1.ResourceCPU]; found {
181                         burstablePodCPURequest += request.MilliValue()
182                 }
183         }
184
185         // make sure best effort is always 2 shares
186         bestEffortCPUShares := uint64(MinShares)
187         configs[v1.PodQOSBestEffort].ResourceParameters.CpuShares = &bestEffortCPUShares
188
189         // set burstable shares based on current observe state
190         burstableCPUShares := MilliCPUToShares(burstablePodCPURequest)
191         configs[v1.PodQOSBurstable].ResourceParameters.CpuShares = &burstableCPUShares
192         return nil
193 }
```


### Memory Handling

Memory is quite a bit different, and is only handled pending the QoS feature gate (alpha as of 1.11, default is disabled), as seen [here](https://github.com/kubernetes/kubernetes/blob/c47ab33e670cd5589b07418457e39b25cea6512f/pkg/kubelet/cm/qos_container_manager_linux.go#L295-L324):
```go
	if utilfeature.DefaultFeatureGate.Enabled(kubefeatures.QOSReserved) {
		for resource, percentReserve := range m.qosReserved {
			switch resource {
			case v1.ResourceMemory:
				m.setMemoryReserve(qosConfigs, percentReserve)
			}
		}

		updateSuccess := true
		for _, config := range qosConfigs {
			err := m.cgroupManager.Update(config)
			if err != nil {
				updateSuccess = false
			}
		}
		if updateSuccess {
			klog.V(4).Infof("[ContainerManager]: Updated QoS cgroup configuration")
			return nil
		}

		// If the resource can adjust the ResourceConfig to increase likelihood of
		// success, call the adjustment function here.  Otherwise, the Update() will
		// be called again with the same values.
		for resource, percentReserve := range m.qosReserved {
			switch resource {
			case v1.ResourceMemory:
				m.retrySetMemoryReserve(qosConfigs, percentReserve)
			}
		}
	}
```

The actual cgroup handling is done pending the QoS [class](https://github.com/kubernetes/kubernetes/blob/c47ab33e670cd5589b07418457e39b25cea6512f/pkg/kubelet/cm/qos_container_manager_linux.go#L195-L241):
```go
// setMemoryReserve sums the memory limits of all pods in a QOS class,
// calculates QOS class memory limits, and set those limits in the
// CgroupConfig for each QOS class.
func (m *qosContainerManagerImpl) setMemoryReserve(configs map[v1.PodQOSClass]*CgroupConfig, percentReserve int64) {
	qosMemoryRequests := map[v1.PodQOSClass]int64{
		v1.PodQOSGuaranteed: 0,
		v1.PodQOSBurstable:  0,
	}

	// Sum the pod limits for pods in each QOS class
	pods := m.activePods()
	for _, pod := range pods {
		podMemoryRequest := int64(0)
		qosClass := v1qos.GetPodQOS(pod)
		if qosClass == v1.PodQOSBestEffort {
			// limits are not set for Best Effort pods
			continue
		}
		req, _ := resource.PodRequestsAndLimits(pod)
		if request, found := req[v1.ResourceMemory]; found {
			podMemoryRequest += request.Value()
		}
		qosMemoryRequests[qosClass] += podMemoryRequest
	}

	resources := m.getNodeAllocatable()
	allocatableResource, ok := resources[v1.ResourceMemory]
	if !ok {
		klog.V(2).Infof("[Container Manager] Allocatable memory value could not be determined.  Not setting QOS memory limts.")
		return
	}
	allocatable := allocatableResource.Value()
	if allocatable == 0 {
		klog.V(2).Infof("[Container Manager] Memory allocatable reported as 0, might be in standalone mode.  Not setting QOS memory limts.")
		return
	}

	for qos, limits := range qosMemoryRequests {
		klog.V(2).Infof("[Container Manager] %s pod requests total %d bytes (reserve %d%%)", qos, limits, percentReserve)
	}

	// Calculate QOS memory limits
	burstableLimit := allocatable - (qosMemoryRequests[v1.PodQOSGuaranteed] * percentReserve / 100)
	bestEffortLimit := burstableLimit - (qosMemoryRequests[v1.PodQOSBurstable] * percentReserve / 100)
	configs[v1.PodQOSBurstable].ResourceParameters.Memory = &burstableLimit
	configs[v1.PodQOSBestEffort].ResourceParameters.Memory = &bestEffortLimit
}
```

## pod cgroup creation

After updating QoS, the pod specific cgroup is created, as seen [here](https://github.com/kubernetes/kubernetes/blob/c47ab33e670cd5589b07418457e39b25cea6512f/pkg/kubelet/kubelet.go#L1616).

Where the cgroup is created depends on the pod's QoS.  Guaranteed are placed in the kubernetes 'root', while burstable and best-effort are placed at root/burstable and root/best-effort, respectively.

Implementation of the function is [here](https://github.com/kubernetes/kubernetes/blob/c47ab33e670cd5589b07418457e39b25cea6512f/pkg/kubelet/cm/pod_container_manager_linux.go#L76-L104)

```go
// EnsureExists takes a pod as argument and makes sure that
// pod cgroup exists if qos cgroup hierarchy flag is enabled.
// If the pod level container doesn't already exist it is created.
func (m *podContainerManagerImpl) EnsureExists(pod *v1.Pod) error {
	podContainerName, _ := m.GetPodContainerName(pod)
	// check if container already exist
	alreadyExists := m.Exists(pod)
	if !alreadyExists {
		// Create the pod container
		containerConfig := &CgroupConfig{
			Name:               podContainerName,
			ResourceParameters: ResourceConfigForPod(pod, m.enforceCPULimits, m.cpuCFSQuotaPeriod),
		}
		if utilfeature.DefaultFeatureGate.Enabled(kubefeatures.SupportPodPidsLimit) && m.podPidsLimit > 0 {
			containerConfig.ResourceParameters.PidsLimit = &m.podPidsLimit
		}
		if err := m.cgroupManager.Create(containerConfig); err != nil {
			return fmt.Errorf("failed to create container for %v : %v", podContainerName, err)
		}
	}
	// Apply appropriate resource limits on the pod container
	// Top level qos containers limits are not updated
	// until we figure how to maintain the desired state in the kubelet.
	// Because maintaining the desired state is difficult without checkpointing.
	if err := m.applyLimits(pod); err != nil {
		return fmt.Errorf("failed to apply resource limits on container for %v : %v", podContainerName, err)
	}
	return nil
}
```

Resource summation for the pod is handled by [`ResourceConfigForPod`](https://github.com/kubernetes/kubernetes/blob/c47ab33e670cd5589b07418457e39b25cea6512f/pkg/kubelet/cm/helpers_linux.go#L109)

<details><summary> ResourceConfigForPod: </summary>
<p>

```go
// ResourceConfigForPod takes the input pod and outputs the cgroup resource config.
func ResourceConfigForPod(pod *v1.Pod, enforceCPULimits bool, cpuPeriod uint64) *ResourceConfig {
	// sum requests and limits.
	reqs, limits := resource.PodRequestsAndLimits(pod)

	cpuRequests := int64(0)
	cpuLimits := int64(0)
	memoryLimits := int64(0)
	if request, found := reqs[v1.ResourceCPU]; found {
		cpuRequests = request.MilliValue()
	}
	if limit, found := limits[v1.ResourceCPU]; found {
		cpuLimits = limit.MilliValue()
	}
	if limit, found := limits[v1.ResourceMemory]; found {
		memoryLimits = limit.Value()
	}

	// convert to CFS values
	cpuShares := MilliCPUToShares(cpuRequests)
	cpuQuota := MilliCPUToQuota(cpuLimits, int64(cpuPeriod))

	// track if limits were applied for each resource.
	memoryLimitsDeclared := true
	cpuLimitsDeclared := true
	// map hugepage pagesize (bytes) to limits (bytes)
	hugePageLimits := map[int64]int64{}
	for _, container := range pod.Spec.Containers {
		if container.Resources.Limits.Cpu().IsZero() {
			cpuLimitsDeclared = false
		}
		if container.Resources.Limits.Memory().IsZero() {
			memoryLimitsDeclared = false
		}
		containerHugePageLimits := HugePageLimits(container.Resources.Requests)
		for k, v := range containerHugePageLimits {
			if value, exists := hugePageLimits[k]; exists {
				hugePageLimits[k] = value + v
			} else {
				hugePageLimits[k] = v
			}
		}
	}

	// quota is not capped when cfs quota is disabled
	if !enforceCPULimits {
		cpuQuota = int64(-1)
	}

	// determine the qos class
	qosClass := v1qos.GetPodQOS(pod)

	// build the result
	result := &ResourceConfig{}
	if qosClass == v1.PodQOSGuaranteed {
		result.CpuShares = &cpuShares
		result.CpuQuota = &cpuQuota
		result.CpuPeriod = &cpuPeriod
		result.Memory = &memoryLimits
	} else if qosClass == v1.PodQOSBurstable {
		result.CpuShares = &cpuShares
		if cpuLimitsDeclared {
			result.CpuQuota = &cpuQuota
			result.CpuPeriod = &cpuPeriod
		}
		if memoryLimitsDeclared {
			result.Memory = &memoryLimits
		}
	} else {
		shares := uint64(MinShares)
		result.CpuShares = &shares
	}
	result.HugePageLimit = hugePageLimits
	return result
}
```

</p>
</details>

The application of the cgroup is handled by a [cgroup manager](https://github.com/kubernetes/kubernetes/blob/c47ab33e670cd5589b07418457e39b25cea6512f/pkg/kubelet/cm/cgroup_manager_linux.go#L448):
```go
447 // Create creates the specified cgroup
448 func (m *cgroupManagerImpl) Create(cgroupConfig *CgroupConfig) error {
449         start := time.Now()
450         defer func() {
451                 metrics.CgroupManagerDuration.WithLabelValues("create").Observe(metrics.SinceInSeconds(start))
452                 metrics.DeprecatedCgroupManagerLatency.WithLabelValues("create").Observe(metrics.SinceInMicroseconds(start))
453         }()
454
455         resources := m.toResources(cgroupConfig.ResourceParameters)
456
457         libcontainerCgroupConfig := &libcontainerconfigs.Cgroup{
458                 Resources: resources,
459         }
460         // libcontainer consumes a different field and expects a different syntax
461         // depending on the cgroup driver in use, so we need this conditional here.
462         if m.adapter.cgroupManagerType == libcontainerSystemd {
463                 updateSystemdCgroupInfo(libcontainerCgroupConfig, cgroupConfig.Name)
464         } else {
465                 libcontainerCgroupConfig.Path = cgroupConfig.Name.ToCgroupfs()
466         }
467
468         if utilfeature.DefaultFeatureGate.Enabled(kubefeatures.SupportPodPidsLimit) && cgroupConfig.ResourceParameters != nil && cgroupConfig.ResourceParameters.PidsLimit != nil {
469                 libcontainerCgroupConfig.PidsLimit = *cgroupConfig.ResourceParameters.PidsLimit
470         }
471
472         // get the manager with the specified cgroup configuration
473         manager, err := m.adapter.newManager(libcontainerCgroupConfig, nil)
474         if err != nil {
475                 return err
476         }
477
478         // Apply(-1) is a hack to create the cgroup directories for each resource
479         // subsystem. The function [cgroups.Manager.apply()] applies cgroup
480         // configuration to the process with the specified pid.
481         // It creates cgroup files for each subsystems and writes the pid
482         // in the tasks file. We use the function to create all the required
483         // cgroup files but not attach any "real" pid to the cgroup.
484         if err := manager.Apply(-1); err != nil {
485                 return err
486         }
487
488         // it may confuse why we call set after we do apply, but the issue is that runc
489         // follows a similar pattern.  it's needed to ensure cpu quota is set properly.
490         m.Update(cgroupConfig)
491
492         return nil
493 }
```
