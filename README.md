# Agent_loadtest

### Sandboxset monitor
**replicas**: 1k(claim batch size:100,500),10k(claim batch size:500,5k),100k(claim batch size:5k,50k)
**metrics**(sandbox status:): 
- `sandboxset_replicas`(gauge) = creating+available+running+paused
- speed of creating = rate(`sandboxset_sandboxes_created_total{namespace,name}`)
- speed of available = rate(`sandboxset_sandboxes_available_total{namespace, name}`)需要新加
- 失败率:`sandbox_creation_total{failure}`/`sandboxset_sandboxes_created_total`
- 缺口1: spec.Replicas - `sandboxset_desired_replicas`
- 缺口2: spec.Replicas - `sandboxset_updated_available_replicas`
- rolling update: 更新完所有的总时间
- 最新版本的sandbox:`sandboxset_updated_replicas`,单个duration需要加上?
- 最新版本的更新速度：需要新加：`sandboxset_sandboxes_updated_total{namespace, name} `
- sandboxset_claims_total: 针对该 set 的 claim 操作累计，`sandboxset_sandboxes_claimed_total{namespace, name}`,只有cr路径，需要加e2b


duration:sandbox_creation_duration_seconds(creation→ready 的时长直方图)。
sandbox_claim_creation_responses{namespace,result}

Counter	在哪	含义
sandboxset_sandboxes_created_total{namespace,name}	sandboxset 控制器	补池创建累计
sandboxset_sandboxes_claimed_total{namespace,name}	sandboxset 控制器(source=k8s)	被 claim 累计(仅 CR 路径)
sandboxset_claims_total	sandboxclaim/core	针对该 set 的 claim 操作累计
缺的(要补):sandboxset_sandboxes_deleted_total(删除/缩容)、sandboxset_sandboxes_updated_total(更新)。

所以现成 counter 就 created / claimed(+ claims_total),删除和更新没有。

- speed of available = rate(`sandbox_creation_total{namespace,result}`)(第一次变ready)，不是某个set是所有的
in place: rate(`sandbox_inplace_update_duration_seconds_count`),*sandbox_inplace_update_duration_seconds(从控制器收到通知先打false,k8sresize,打 	true)* 也是所有的

### E2E e2b leval(sandbox-manager level)

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

### Reconcile sandbox-controller level

- workqueue depth:
  
**SandboxSet 控制器(管池子)**
- **scale_up*	
- **scale_down*
- rolling_update（单轮，MaxUnavailable 为上限）
- gc_dead	清理 dead sandbox
- status_sync	子 sandbox 变了,只重算状态

**Sandbox 控制器(管单个实例)**
- create_pod	Pending,无 Pod → 建 Pod
- sync_ready	Pod ready → 置 Running/Ready
- pause	删 Pod
- resume	重建 Pod + 初始化
- upgrade	重建升级
- inplace_update	原地改
- terminating	删除/finalizer

**SandboxClaim 控制器(管批量 claim)**
- claim_batch	抢一批 sandbox
- completed	转完成态
- expired	TTL 到期处理

**SandboxUpdateOps** 控制器(管批量升级已 claim 的)
⚠️ 这个现在零指标,内部分支我没细读,埋点时要先确认。大致:
- update_batch	推进一批更新
- rolling / partition	按策略进度
- completed	完成

### k8s side(sandbox level)
单个 sandbox 的 Pod,k8s 自己做的事(不是 OpenKruise reconcile):

阶段	k8s 组件	指标	kwok 下
写 Sandbox/Pod 落库	apiserver + etcd	apiserver_request_duration、etcd fsync/commit、db size	真
调度 Pod 到节点	scheduler	scheduler_e2e_scheduling_duration	真(kwok 假节点)
起 Pod → Running	kubelet / kwok	(kwok 模拟)	假
Pod Ready	kubelet / kwok	PodReady condition 时间戳	假(kwok stage 时序)

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
