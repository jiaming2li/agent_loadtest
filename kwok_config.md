
### Pause

#### Manager side
`sandbox.go:311-364`

```
if err = pauseTask.Wait(time.Minute); err != nil {
	log.Error(err, "failed to wait sandbox pause")
	return err
}
```
`pauseTask.Wait(time.Minute)`: 等待sandbox到达 paused 状态(SandboxPaused 条件=True / phase=Paused)（最多 1min）



#### Controller side   
`spec.Paused=true` 触发 reconcile:`common_control.go: EnsureSandboxPaused(), 191-247`


```
if rejected := r.checkpointControl.AssumePodCheckpointed(ctx, pod, box, newStatus, cond); rejected {
	return nil
}

err := client.IgnoreNotFound(r.Delete(ctx, pod, &client.DeleteOptions{GracePeriodSeconds: ptr.To(int64(5))}))
if err != nil {
```
- r.Delete()
- 假 pod 要有真实的 containerStatuses(imageID),否则 AssumePodCheckpointed 可能记录不全。别在 kwok 上开 SandboxPauseCheckpoint(Alpha,默认关)


4. 下次 reconcile：pod == nil → Paused 条件 = True(reason=DeletePod)
     → phase = Paused，完成



### Resume

#### Manager side
`sandbox.go:373-461`

```
if err = resumeTask.Wait(resumeWaitMaxTimeout); err != nil {
	if ctxErr := ctx.Err(); ctxErr != nil {
		log.Info("stop waiting sandbox resume: request canceled by client (disconnected or client-side timeout)",
			"err", err, "ctxErr", ctxErr)
		return err
	}
	log.Error(err, "failed to wait sandbox resume")
	return err
}
```
`resumeTask.Wait(resumeWaitMaxTimeout)`: 等待sandbox到达 running 状态(resume = 重建 pod + 等 ready + 重做 runtime/CSptrI 初始化)（最多 1min）



#### Controller side   
`spec.Paused=false` 触发 reconcile:`common_control.go: EnsureSandboxResumed(), 249-299`


```
var err error
if pod == nil {
	delta := r.checkpointControl.GetPodTemplateDelta(ctx, box)
	_, err = r.podControl.CreatePod(ctx, CreatePodArgs{Box: box, NewStatus: newStatus, PodTemplateDelta: delta})
	return err
}

if pod.Status.Phase == corev1.PodRunning && isContainersConsistent(pod, box) {}
```

- pod phase 变 running → EnsureSandboxResumed(267 行)→ sandbox phase=Running + RuntimeInitialized=Pending
- pod condition 变 Ready(容器通过 readiness probe) → 【Pod update 事件】→ sandbox reconcile → EnsureSandboxUpdated：pod ready → Initialize → RuntimeInitialized=True

```
// If RuntimeInitialized is pending (set during resume), wait for Pod Ready then run Initialize
	initCond := utils.GetSandboxCondition(newStatus, string(agentsv1alpha1.RuntimeInitialized))
	if initCond != nil && initCond.Status != metav1.ConditionTrue {
		pCond := utils.GetPodCondition(&pod.Status, corev1.PodReady)
		if pCond == nil || pCond.Status != corev1.ConditionTrue {
			klog.InfoS("Waiting for pod ready before initialization", "sandbox", klog.KObj(box))
			return nil
		}
		if err := r.initializer.Initialize(ctx, box, newStatus); err != nil {
			return err
		}
	}

	// For non-Recreate upgrade policy (e.g., sandbox-manager triggered inplace update via annotation),
	// perform inplace update directly without entering the full upgrade lifecycle (PreUpgrade -> UpgradePod -> PostUpgrade).
	if box.Spec.UpgradePolicy == nil || box.Spec.UpgradePolicy.Type != agentsv1alpha1.SandboxUpgradePolicyRecreate {
		done, err := r.handleInplaceUpdateSandbox(ctx, args)
		if err != nil {
			return err
		} else if !done {
			return nil
		}
	}
	syncSandboxStatusFromPod(pod, newStatus)
	return nil
```
sandbox_initializer.go:46-79:
① 设 Pending：真（status 写）
② 等 Pod Ready：kwok stage 设 PodReady=True → 满足
③ Initialize：
   - 无 runtime/CSI 注解（skip-init-runtime claim）→ reinitRuntime no-op、CSI 跳过
     → 直接 RuntimeInitialized=True → 通 ✅
   - 有注解 → 连 envd/CSI → kwok 做不了 → Failed → 卡 ❌

  


### Claim


### Inplace-update
