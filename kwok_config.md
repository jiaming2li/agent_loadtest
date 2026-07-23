
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

sets `e2b.agents.kruise.io/skip-init-runtime": "true`; avoid adding CSI metadata and identity; keeps `SecurityIdentityProvider` as deflaut(false).

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
  createOnNoStock: true        # optional：create new sdx when pool is empty
```
- E2B:
```
META = {
    "e2b.agents.kruise.io/skip-init-runtime": "true",
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
    client.Client                          // ← embed k8s client（visit apiserver）
    generatePatchBodyFunc GeneratePatchBodyFunc  // generate image/metadata patch body
    useDirectResourcePatch atomic.Bool     
}
```

potential blocking:

`control.Update` call apiserver to change `spec.image`(+hash); kubelet watches the change, pull new image, restart container and write `imageID` back. 

solution:

Stage fakes kubelet by calling apiserver to change `status.imageID`  as 你的 controller 的完成判定(IsInplaceUpdateCompleted 那套)就是靠"status.imageID 变没变"来确认升级真的生效了。


## Rolling update
batch delete+create, similar to `pause` and `resume`


## Pause
  
manager changing `spec.Paused=true` invoke reconcile:`common_control.go: EnsureSandboxPaused(), 191-247`


```
if rejected := r.checkpointControl.AssumePodCheckpointed(ctx, pod, box, newStatus, cond); rejected {
	return nil
} //before deleting pod, make usre pod checkpointed

func (c *CheckpointControl) AssumePodCheckpointed() bool {
	if !utilfeature.DefaultFeatureGate.Enabled(features.SandboxPauseCheckpointGate) {
		return false
	}
```

```
err := client.IgnoreNotFound(r.Delete(ctx, pod, &client.DeleteOptions{GracePeriodSeconds: ptr.To(int64(5))}))
if err != nil {
```


potential blocking:

- making Checkpoint needs real pod.
- `r.Delete()`, apiserver set pod `deletionTimestamp`，kubelet watch `deletionTimestamp`, stop container, call apiserver remove pod obj from etcd.

solution:
- keep `SandboxPauseCheckpoint` as default(false)
- pod-delete Stage:`selector`: `deletionTimestamp` Exists(match deleting pod)+ next: { delete: true }(let kwok-controller send apiserver DELETE, apiserver remove pod from etcd).



## Resume
  
manager changing `spec.Paused=false` invoke reconcile:`common_control.go: EnsureSandboxResumed(), 249-299`

```
if pod == nil {
	delta := r.checkpointControl.GetPodTemplateDelta(ctx, box)
	_, err = r.podControl.CreatePod(ctx, CreatePodArgs{Box: box, NewStatus: newStatus, PodTemplateDelta: delta})
	return err
}
```
- call apiserver create pod: apiserver write into etcd → kube-scheduler find a node → node's kubelet pull image, build container, run readiness probe(in kubelet) → APIserver write status running and condition ready
- pod phase becomes running → EnsureSandboxResumed(267)→ sandbox phase=Running + RuntimeInitialized=Pending
- pod condition become Ready(container pass readiness probe) → Pod update event → sandbox reconcile → EnsureSandboxUpdated：pod ready → Initialize → RuntimeInitialized=True

```
if initCond != nil && initCond.Status != metav1.ConditionTrue {#RuntimeInitialized=Pending
	if err := r.initializer.Initialize(ctx, box, newStatus); err != nil {
		return err
	}
}
```

```
if len(csiMountConfigRequests) != 0 {
	duration, mountErr := utilruntime.ProcessCSIMounts(ctx, sbxForInit, config.CSIMountOptions{
	})
}

if initRuntimeOpts != nil {
	if _, err = utilruntime.InitRuntime(ctx, sbxForInit, *initRuntimeOpts, nil); err != nil {
	}
}

if !identity.IsIdentityProviderRequested(sbxForInit) {
	return nil
}
```

solutions:
- kwok call APIserver write status running and condition ready
- keeps `e2b.agents.kruise.io/skip-init-runtime": "true`; avoid adding CSI metadata and identity; keeps `SecurityIdentityProvider` as deflaut to avoid InitRuntime, SecurityToken and CSI processing.


