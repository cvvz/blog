---
title: "为什么删除Pod时webhook收到三次delete请求"
date: 2020-12-13T19:26:15+08:00
draft: false
comments: true
keywords: ["kubernetes"]
tags: ["kubernetes"]
toc: true
autoCollapseToc: false
---

最近在玩admission webhook时，发现一个奇怪的现象：我配置了validatingWebhookConfiguration使其监听pod的删除操作，结果发现每次删除Pod的时候，webhook会收到三次delete请求：

{{< figure src="/3-delete.png" width="1000px" >}}

从日志打印上可以分析出，第一次删除请求来自于kubectl客户端，后面两次来自于pod所在的node节点。为什么会收到三次delete请求呢？

## 删除一个Pod的过程

通过阅读kube-apiserver和kubelet源码，我把一个pod的删除过程总结成如下这幅流程图，三个红色加粗的请求即为webhook收到的三次delete请求。
{{< figure src="/delete-pod.drawio.svg" width="800px">}}

### kube-apiserver处理第一次删除请求

首先，由kubectl发来的delete请求，会经过kube-apiserver的admission-controller进行准入校验。我们定义了admission webhook，所以kube-apiserver会将该请求相关的信息封装在**AdmissionReview**结构体中发送给webhook。这是第一次webhook收到delete请求。

kube-apiserver作为一个http服务器，它的handler在`staging/src/k8s.io/apiserver/pkg/endpoints/installer.go`文件中的`registerResourceHandlers`函数中定义。其中`DELETE`请求的handler是`restfulDeleteResource`：

```go
case "DELETE": // Delete a resource.
    // ...

    handler := metrics.InstrumentRouteFunc(action.Verb, group, version, resource, subresource, requestScope, metrics.APIServerComponent, deprecated, removedRelease, restfulDeleteResource(gracefulDeleter, isGracefulDeleter, reqScope, admit))

    ...
```

`restfulDeleteResource`调用`DeleteResource`，后者则调用`staging/src/k8s.io/apiserver/pkg/registry/generic/registry/store.go`文件中的`Delete`方法对对象进行删除

```go
func restfulDeleteResource(r rest.GracefulDeleter, allowsOptions bool, scope handlers.RequestScope, admit admission.Interface) restful.RouteFunction {
	return func(req *restful.Request, res *restful.Response) {
		handlers.DeleteResource(r, allowsOptions, &scope, admit)(res.ResponseWriter, req.Request)
	}
}
```

```go
func DeleteResource(r rest.GracefulDeleter, allowsOptions bool, scope *RequestScope, admit admission.Interface) http.HandlerFunc {
    //...

    trace.Step("About to delete object from database")
		wasDeleted := true
		userInfo, _ := request.UserFrom(ctx)
		staticAdmissionAttrs := admission.NewAttributesRecord(nil, nil, scope.Kind, namespace, name, scope.Resource, scope.Subresource, admission.Delete, options, dryrun.IsDryRun(options.DryRun), userInfo)
		result, err := finishRequest(timeout, func() (runtime.Object, error) {
			obj, deleted, err := r.Delete(ctx, name, rest.AdmissionToValidateObjectDeleteFunc(admit, staticAdmissionAttrs, scope), options)
			wasDeleted = deleted
			return obj, err
		})
		if err != nil {
			scope.err(err, w, req)
			return
		}
        trace.Step("Object deleted from database")
        
        ...
}
```

`Delete`方法中，在`BeforeDelete`函数中判断是否需要优雅删除，判断的标准是`DeletionGracePeriodSeconds`值是否为0，不为零则认为是优雅删除，kube-apiserver不会立即将这个API对象从etcd中删除，否则直接删除。

对于Pod而言，默认`DeletionGracePeriodSeconds`为30秒，因此这里不会被kube-apiserver立刻删除掉。而是将`DeletionTimestamp`设置为当前时间，`DeletionGracePeriodSeconds`设置为默认值30秒。

### kubelet杀掉容器

kube-apiserver设置好`DeletionTimestamp`和`DeletionGracePeriodSeconds`这两个字段后，kubelet 会watch到Pod的更新。那kubelet list-watch机制又是怎么实现的呢？

Kubelet在`makePodSourceConfig`函数中，监听了三种类型的Pod：通过[文件系统上的配置文件](https://kubernetes.io/zh/docs/tasks/configure-pod-container/static-pod/#configuration-files)配置的静态Pod，通过[web 网络上的配置文件](https://kubernetes.io/zh/docs/tasks/configure-pod-container/static-pod/#pods-created-via-http)配置的静态Pod，以及kube-apiserver中的pod。我们主要关心第三种。

Kubelet通过reflactor watch到Pod资源发生变化后，是通过channel的方式将Pod及其变化传递给syncLoop主控制循环中进行处理的，**并没有使用informer+workqueque的方式**。

kubelet的主控制循环在`pkg/kubelet/kubelet.go`文件中的`syncLoopIteration`函数：

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

        ...
```

当Pod的`DeletionTimestamp`被设置时，Kubelet会走入`kubetypes.DELETE`这个分支，最终会调用到`pkg/kubelet/kubelet.go`中的`syncPod`函数，**`syncPod` 这个函数是 kubelet 核心处理函数**。这个函数会调用到容器运行时的`KillPod`方法，该方法进而又会以goroutine的方式，使用`pkg/kubelet/kuberuntime/kuberuntime_container.go`中定义的`killContainer`方法**并行的杀掉**所有容器。`killContainer`的代码实现如下所示：

```go
func (m *kubeGenericRuntimeManager) killContainer(pod *v1.Pod, containerID kubecontainer.ContainerID, containerName string, message string, gracePeriodOverride *int64) error {
	...

	// From this point, pod and container must be non-nil.
	gracePeriod := int64(minimumGracePeriodInSeconds)
	switch {
	case pod.DeletionGracePeriodSeconds != nil:
		gracePeriod = *pod.DeletionGracePeriodSeconds
	case pod.Spec.TerminationGracePeriodSeconds != nil:
		gracePeriod = *pod.Spec.TerminationGracePeriodSeconds
	}

	if len(message) == 0 {
		message = fmt.Sprintf("Stopping container %s", containerSpec.Name)
	}
	m.recordContainerEvent(pod, containerSpec, containerID.ID, v1.EventTypeNormal, events.KillingContainer, message)

	// Run internal pre-stop lifecycle hook
	if err := m.internalLifecycle.PreStopContainer(containerID.ID); err != nil {
		return err
	}

	// Run the pre-stop lifecycle hooks if applicable and if there is enough time to run it
	if containerSpec.Lifecycle != nil && containerSpec.Lifecycle.PreStop != nil && gracePeriod > 0 {
		gracePeriod = gracePeriod - m.executePreStopHook(pod, containerID, containerSpec, gracePeriod)
	}
	// always give containers a minimal shutdown window to avoid unnecessary SIGKILLs
	if gracePeriod < minimumGracePeriodInSeconds {
		gracePeriod = minimumGracePeriodInSeconds
	}
	if gracePeriodOverride != nil {
		gracePeriod = *gracePeriodOverride
		klog.V(3).Infof("Killing container %q, but using %d second grace period override", containerID, gracePeriod)
	}

	klog.V(2).Infof("Killing container %q with %d second grace period", containerID.String(), gracePeriod)

	err := m.runtimeService.StopContainer(containerID.ID, gracePeriod)
	if err != nil {
		klog.Errorf("Container %q termination failed with gracePeriod %d: %v", containerID.String(), gracePeriod, err)
	} else {
		klog.V(3).Infof("Container %q exited normally", containerID.String())
	}

	m.containerRefManager.ClearRef(containerID)

	return err
}  
```
这个方法就是先调用prestop hook，然后在通过`runtimeService.StopContainer`方法杀掉容器进程，整个过程总时长不能超过`DeletionGracePeriodSeconds`。注意，prestop hook是不会进行重试的，失败了kubelet也不管，容器还是照杀不误。

### statusManager发送删除请求

kubelet以goroutine的方式运行着一个`statusManager`，它的作用就是周期性的监听Pod的状态变化，然后执行`func (m *manager) syncPod(uid types.UID, status versionedPodStatus) {`。在`syncPod`中，注意到有如下的逻辑：

```go
func (m *manager) syncPod(uid types.UID, status versionedPodStatus) {
    ...

    if m.canBeDeleted(pod, status.status) {
		deleteOptions := metav1.DeleteOptions{
			GracePeriodSeconds: new(int64),
			// Use the pod UID as the precondition for deletion to prevent deleting a
			// newly created pod with the same name and namespace.
			Preconditions: metav1.NewUIDPreconditions(string(pod.UID)),
		}
		err = m.kubeClient.CoreV1().Pods(pod.Namespace).Delete(context.TODO(), pod.Name, deleteOptions)
		if err != nil {
			klog.Warningf("Failed to delete status for pod %q: %v", format.Pod(pod), err)
			return
		}
		klog.V(3).Infof("Pod %q fully terminated and removed from etcd", format.Pod(pod))
		m.deletePodStatus(uid)
	}
}
```

也就是说，**`statusManager`发现Pod可以被删除的时候，就会去调用clientset的delete接口将Pod资源从kube-apiserver中删掉**。那什么时候Pod可以被删除呢？自然是在上一步中，kubelet将Pod的容器、卷、cgroup sandbox等资源统统删除掉，就可以被删除了。

这里，webhook就会收到第二次删除请求，而且这次请求中，将`GracePeriodSeconds`设置为了0，这就代表着kube-apiserver收到这个DELETE请求后，可以将Pod从etcd中删除了。

### 第三次delete请求

webhook为什么会收到第三次delete请求，这个问题着实困扰了我很久。

从日志的serviceAccount的信息来看，很像是节点上的组件又发了一次DELETE请求。是kubelet吗？还是kube-proxy？但是查看相关日志和代码，没有发现任何可疑点。

其实，第三次DELETE请求是kube-apiserver自己发的。

在第一部分中，我提到kube-apiserver收到DELETE请求后最终会调用`staging/src/k8s.io/apiserver/pkg/registry/generic/registry/store.go`文件中的`Delete`方法，然后由于走的是优雅删除，它更新完Pod的`DeletionTimestamp`和`DeletionGracePeriodSeconds`两个字段后，就返回了。

现在，第二次DELETE请求将`GracePeriodSeconds`设置为了0，于是现在可以开始执行实际的删除操作了。

```go
func (e *Store) Delete(ctx context.Context, name string, deleteValidation rest.ValidateObjectFunc, options *metav1.DeleteOptions) (runtime.Object, bool, error) {
    ...
    // delete immediately, or no graceful deletion supported
    klog.V(6).Infof("going to delete %s from registry: ", name)
    out = e.NewFunc()
    if err := e.Storage.Delete(ctx, key, out, &preconditions, storage.ValidateObjectFunc(deleteValidation), dryrun.IsDryRun(options.DryRun)); err != nil {
        // Please refer to the place where we set ignoreNotFound for the reason
        // why we ignore the NotFound error .
        if storage.IsNotFound(err) && ignoreNotFound && lastExisting != nil {
            // The lastExisting object may not be the last state of the object
            // before its deletion, but it's the best approximation.
            out, err := e.finalizeDelete(ctx, lastExisting, true)
            return out, true, err
        }
        return nil, false, storeerr.InterpretDeleteError(err, qualifiedResource, name)
    }
    ...
}
```

在`e.Storage.Delete`方法中，定义了`storage.ValidateObjectFunc(deleteValidation)`参数，仔细阅读这个方法的实现细节，原来，kube-apiserver在进行删除前，还会再对这个删除操作执行一次准入控制校验，即Validating和Mutating。代码逻辑见`staging/src/k8s.io/apiserver/pkg/storage/etcd3/store.go`中的`conditionalDelete`函数：

```go
func (s *store) conditionalDelete(ctx context.Context, key string, out runtime.Object, v reflect.Value, preconditions *storage.Preconditions, validateDeletion storage.ValidateObjectFunc) error {
    ...
    for {
		origState, err := s.getState(getResp, key, v, false)
		if err != nil {
			return err
		}
		if preconditions != nil {
			if err := preconditions.Check(key, origState.obj); err != nil {
				return err
			}
		}
		if err := validateDeletion(ctx, origState.obj); err != nil {
			return err
		}
		startTime := time.Now()
		txnResp, err := s.client.KV.Txn(ctx).If(
			clientv3.Compare(clientv3.ModRevision(key), "=", origState.rev),
		).Then(
			clientv3.OpDelete(key),
		).Else(
			clientv3.OpGet(key),
        ).Commit()
    ...

}
```

validateDeletion 即为进行DELETE准入控制校验的地方，这个过程中必定会调用到Validating webhook，也就有了第三次delete请求。至于为什么要再做一次准入控制，我也不太明白。
