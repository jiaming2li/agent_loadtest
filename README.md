# agent_loadtest

1000sandbox

## Tested func
How to decide start and end?
### Sandboxset 
**Metrics**  

- sdx creation speed
```
# after call k8s apiserver to create sandbox crd, count ++
sandboxset_controller.go 354
if err := r.Create(ctx, sbx); err != nil {
		r.Recorder.Eventf(sbs, corev1.EventTypeWarning, EventCreateSandboxFailed, "Failed to create sandbox: %s", err)
		return nil, err
	}
	sandboxSetSandboxesCreatedTotal.WithLabelValues(sbs.Namespace, sbs.Name).Inc()

sum(rate(sandboxset_sandboxes_created_total[1m]))                       # 全局每秒新建
rate(sandboxset_sandboxes_created_total{namespace="ns",name="my-sbs"}[1m])  # 单个 SandboxSet
```
- 单个 sandbox 预热要多久(延迟分布) 
```
# event_handler.go 68-79
now: oldState == agentsv1alpha1.SandboxStateCreating && newState == agentsv1alpha1.SandboxStateAvailable
cond.LastTransitionTime.Time # copy time when pod ready condition changes
newSbx.CreationTimestamp.Time # time when k8s receive request of creating sdx crd
```
```
SandboxStateAvailable   // 被 SBS 管 + sandbox Readycondition is true  → Available
SandboxStateCreating    // 被 SBS 管 + 没Ready → Creating

```

### Claim Sandbox
#### Claiming Sandboxes via E2B SDK
 
**Metrics**: 
- `retries`: number of retries of claiming
- `wait`: time waiting for idle claim worker 
- `PickAndLock`:选 sandbox + 改对象 + 加锁 三步合一(Step1+Step2):pickAnAvailableSandbox + modifyPickedSandbox + performLockSandbox。含一次 apiserver Update 写锁。
- `LockType`:creat/update/speculate
- `WaitReady`: time waiting for being ready(creat and speculate)
- `RetryCost`: 浪费在失败尝试上的时间。只有当某次 TryClaimSandbox 失败时,才把它的 Total 累加进来。


#### Claiming Sandboxes via SandboxClaim

#### Batch Claiming Sandboxes

#### Creating Sandbox Directly from Template(baseline? guarentee same config,use snapshot?)

### pause/resume
#### pause   
pause:  删 Pod（释放 CPU/内存等算力），但保留 sandbox 对象 + 持久化状态      
resume: 重建 Pod + 恢复状态     
持久化卷(VolumeClaimTemplates,见 sandbox_controller.go:180 ensureVolumeClaimTemplates 注释「for persistent data recovery during sleep/wake」);
**metric**:  
***controller level*: `source="k8s"`(真实生命周期)  
`sandbox_status_unpaused`:pause 完成(True)后变 0;resume 后 condition 被删,这个 gauge 不再被更新,可能停在旧值直到 deleteSandboxMetrics 清掉(staleness 坑)。   
***sandbox-manager*: `source = e2b`(api) 


**baseline**: no

#### resume
**metric**: latency from func call to sandbox's status become running  
**baseline**: Creating Sandbox Directly from Template

**method**:



### Image Replacement/resource adjustment
#### Upgrade Pre-warmed Pool Sandboxes (SandboxSet)
**metric**: speed of updating
**baseline**: 
**method**: watch version changing

#### Upgrade Claimed Sandboxes (SandboxUpdateOps)
