# agent_loadtest

#### sandboxset monitor
**replicas**: 1k(claim batch size:100,500),10k(claim batch size:500,5k),100k(claim batch size:5k,50k)
**metrics**(sandbox status:): 
- gauge(available?)=
- speed of creating = rate(sandboxset_sandboxes_created_total{namespace,name})(r.Create(Sandbox))
- speed of available = rate(sandbox_creation_total{namespace,result})(第一次变ready)
- 失败率       → sandbox_creation_total{failure}/replica
- (replica-gauge)/replica?
- speed of update:
  in place: rate(sandbox_inplace_update_duration_seconds_count),sandbox_inplace_update_duration_seconds(从控制器收到通知先打false,k8sresize,打 	true)
  rolling: update(更新完所有的总时间),（deriv(sandboxset_updated_replicas[1m])）,duration需要加上


duration:sandbox_creation_duration_seconds(creation→ready 的时长直方图)。
sandbox_claim_creation_responses{namespace,result}
#### E2E e2b leval(sandbox-manager level)

```
results = [] 

def create(template): return measure("create", lambda: Sandbox(template=template))
def pause(sbx):       return measure("pause",  lambda: sbx.pause())
def resume(sbx):      return measure("resume", lambda: sbx.connect())
def delete(sbx):      return measure("delete", lambda: sbx.kill())

def measure(op, fn):
    start = time.monotonic()
    ok, err, ret = True, None, None
    try:
        ret = fn()
    except Exception as e:
        ok, err = False, repr(e)
    end = time.monotonic()
    results.append((op, start, end - start, ok, err))
    return ret

```

#### reconcile sandbox-controller level
- sandboxset create:
- sandbox update:
- workqueue depth:
SandboxSet 控制器(管池子)
type	触发	轻重
scale_up	数量不够,补池建新	中
scale_down	数量多了,删	中
rolling_update	版本变了,删旧建新	重
gc_dead	清理 dead sandbox	轻
status_sync	子 sandbox 变了,只重算状态	轻
noop	没事 / 等 expectation	极轻
Sandbox 控制器(管单个实例)
type	触发	轻重
create_pod	Pending,无 Pod → 建 Pod	中
sync_ready	Pod ready → 置 Running/Ready	轻
pause	删 Pod	中
resume	重建 Pod + 初始化	重
upgrade	重建升级	重
inplace_update	原地改	中
terminating	删除/finalizer	中
noop	没事	极轻
SandboxClaim 控制器(管批量 claim)
type	触发	轻重
claim_batch	抢一批 sandbox	重
completed	转完成态	轻
expired	TTL 到期处理	轻
noop	等 / 没事	极轻
SandboxUpdateOps 控制器(管批量升级已 claim 的)
⚠️ 这个现在零指标,内部分支我没细读,埋点时要先确认。大致:

type	触发
update_batch	推进一批更新
rolling / partition	按策略进度
completed	完成
noop	等 / 没事


#### k8s side(sandbox level)

#### 
100ksandbox

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
