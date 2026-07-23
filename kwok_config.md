
## Claim

potential blocking:

`pkg/sandbox-manager/infra/sandboxcr/claim.go`: `runClaimPostProcesses()`

```
if opts.InitRuntime != nil {		
	metrics.InitRuntime, err = runtime.InitRuntime(ctx, sbx.Sandbox, *opts.InitRuntime, sbx.refreshFunc())
}

if identity.IsIdentityProviderRequested(sbx.Sandbox) {
	metrics.SecurityToken, err = identity.ProcessSandboxToken(ctx, cache.GetClient(), sbx.Sandbox)
}

if opts.CSIMount != nil {
	metrics.CSIMount, err = runtime.ProcessCSIMounts(ctx, sbx.Sandbox, *opts.CSIMount)
}
```
InitRuntime, process SecurityToken and CSI are all operated in real pod.


solution:

sets `e2b.agents.kruise.io/skip-init-runtime": "true`; avoid adding CSI metadata and identity; keeps `SecurityIdentityProvider` as deflaut
to avoid InitRuntime, SecurityToken and CSI processing.

- cr:
```
apiVersion: agents.kruise.io/v1alpha1
kind: SandboxClaim
metadata:
  name: batch-claim
  namespace: default
spec:
  templateName:
  replicas: 100
  skipInitRuntime: true
  createOnNoStock: true        # 可选：池子空时补建
```
- E2B:
```
META = {
    "e2b.agents.kruise.io/skip-init-runtime": "true",
    "e2b.agents.kruise.io/claim-timeout-seconds": "8",
    "e2b.agents.kruise.io/wait-ready-timeout-seconds": "5",
}

sbx = Sandbox.create(template=TEMPLATE, timeout=60, request_timeout=25, metadata=META)
```



## Inplace-update
`.spec.template.spec.containers[].image` changing invokes `pkg/controller/sandbox/core/common_inplace_update_handler.go`:
```
control := handler.GetInPlaceUpdateControl()
changed, err := control.Update(ctx, opts)
```
```
type InPlaceUpdateControl struct {
    client.Client                          // ← embed k8s 客户端（访问 apiserver）
    generatePatchBodyFunc GeneratePatchBodyFunc  // 生成 image/metadata 的 patch body
    useDirectResourcePatch atomic.Bool     // K8s<1.33 无 resize 子资源时的兼容 fallback
}
```

potential blocking:

`control.Update` asks apiserver to change `spec.image`(+hash); kubelet watches the change, pull new image, restart container and write `imageID` back. There is no kubelet in fake node.

solution:

in-place-update Stage fakes the kubelet by flipping imageID to a sentinel once a pod is being in-place-updated. Add `delay` + `jitterDurationMilliseconds` for realistic pull/restart timing; add a second, weighted Stage that sets `ImagePullBackOff` and keeps imageID unchanged (with a waiting-reason exclusion on both Stages to prevent re-firing) to model a failure rate. 


## Rolling update
delete+create


## Pause
  
manager changing `spec.Paused=true` invoke reconcile:`common_control.go: EnsureSandboxPaused(), 191-247`


```
if rejected := r.checkpointControl.AssumePodCheckpointed(ctx, pod, box, newStatus, cond); rejected {
	return nil
} //删 pod 前先确保 pod 已 checkpoint
```

```
err := client.IgnoreNotFound(r.Delete(ctx, pod, &client.DeleteOptions{GracePeriodSeconds: ptr.To(int64(5))}))
if err != nil {
```

```
func (c *CheckpointControl) AssumePodCheckpointed(ctx context.Context, pod *corev1.Pod, box *agentsv1alpha1.Sandbox, newStatus *agentsv1alpha1.SandboxStatus, cond *metav1.Condition) bool {
	if !utilfeature.DefaultFeatureGate.Enabled(features.SandboxPauseCheckpointGate) {
	//SandboxPauseCheckpoint 通过 agent-sandbox-controller 的 --feature-
		return false
	}
```

potential blocking:

- making Checkpoint needs real pod.
- `r.Delete()`, apiserver set pod `deletionTimestamp`，kubelet watch `deletionTimestamp`, stop container, apiserver remove pod obj from etcd.

solution:
- keep `SandboxPauseCheckpoint` as default(false)
- pod-delete Stage:`selector`: `deletionTimestamp` Exists(match deleting pod)+ next: { delete: true }(remove pod), add delay(`durationMilliseconds` + `jitterDurationMilliseconds`,如 3~6s) simulate graceshutting and processing time.



## Resume
  
manager changing `spec.Paused=false` invoke reconcile:`common_control.go: EnsureSandboxResumed(), 249-299`

```
if pod == nil {
	delta := r.checkpointControl.GetPodTemplateDelta(ctx, box)
	_, err = r.podControl.CreatePod(ctx, CreatePodArgs{Box: box, NewStatus: newStatus, PodTemplateDelta: delta})
	return err
}
```

- pod phase 变 running → EnsureSandboxResumed(267 行)→ sandbox phase=Running + RuntimeInitialized=Pending
- pod condition 变 Ready(容器通过 readiness probe) → 【Pod update 事件】→ sandbox reconcile → EnsureSandboxUpdated：pod ready → Initialize → RuntimeInitialized=True

```
if initCond != nil && initCond.Status != metav1.ConditionTrue {#RuntimeInitialized=Pending
	if err := r.initializer.Initialize(ctx, box, newStatus); err != nil {
		return err
	}
}
```

solutions:
- 等 Pod Ready：kwok stage 设 PodReady=True → 满足

③ Initialize(sandbox_initializer.go:46-79)
   - 无 runtime/CSI 注解（skip-init-runtime claim）→ reinitRuntime no-op、CSI 跳过
     → 直接 RuntimeInitialized=True → 通 ✅
   - 有注解 → 连 envd/CSI → kwok 做不了 → Failed → 卡 ❌

```
if len(csiMountConfigRequests) != 0 {
	duration, mountErr := utilruntime.ProcessCSIMounts(ctx, sbxForInit, config.CSIMountOptions{
	})
}
```
```
if initRuntimeOpts != nil {
	if _, err = utilruntime.InitRuntime(ctx, sbxForInit, *initRuntimeOpts, nil); err != nil {
	}
}
```

```
if !identity.IsIdentityProviderRequested(sbxForInit) {
	return nil
}
```
