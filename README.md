# OKA_KWOK_Loadtest

## E2E Test(e2b/cr)
### tested functions
#### e2b
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
metrics: p50, p90, p95, tps

#### CR

- rolling update(change `SandboxSet spec.template` → UpdateRevision)  
  metrics:
  - total time
  - speed(最新版本的sandbox:`sandboxset_updated_replicas`,单个duration需要加上?)- 最新版本的更新速度：需要新加：`sandboxset_sandboxes_updated_total{namespace, name} ` 

- in-place update(`SandboxUpdateOps` cr)  
   metrics:
  - total time
  - speed(rate(`sandbox_inplace_update_duration_seconds_count`)) 

  
#### sandboxset metrics
- speed of creating = rate(`sandboxset_sandboxes_created_total{namespace,name}`)
- speed of available = rate(`sandboxset_sandboxes_available_total{namespace, name}`)需要新加
- speed of delet = sandboxset_sandboxes_deleted_total(删除/缩容)
- 失败率:`sandbox_creation_total{failure}`/`sandboxset_sandboxes_created_total`
- 缺口1: spec.Replicas - `sandboxset_desired_replicas`

## Sandbox-manager test

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
metrics: p50, p90, p95


## Sandbox-controller test
### workqueue(different controller)
- workqueue depth
- wait time(): rate(`workqueue_queue_duration_seconds_sum{name="sandboxset"}`[5m])/rate(`workqueue_queue_duration_seconds_count{name="sandboxset"}`[5m])
  

### reconcile   

metrics: p50,p90,p99

**current metrics**:   
- controller reconcile average time: rate(`controller_runtime_reconcile_time_seconds_sum{controller="sandboxset"}`[5m])
  / rate(`controller_runtime_reconcile_time_seconds_count{controller="sandboxset"}`[5m])  

**proposed add**:   

```
func Reconcile(ctx, req) {
    ctx = withAPITimer(ctx)        // 累加器
    start := time.Now()
    ... // 确定 type；client 调用都累加进 ctx
    apiTime := getAPITime(ctx)
    total := time.Since(start)
    reconcileDuration.WithLabelValues(ctrl, type).Observe(total)        // 总
    reconcileAPIDuration.WithLabelValues(ctrl, type).Observe(apiTime)   // apiserver 部分
    // 纯计算 = total − apiTime（也可以直接 Observe 这个）
}
```

**SandboxSet**
- scale_up
- scale_down
- rolling_update（single round，MaxUnavailable）
- gc_dead:	clean dead sandbox
- status_sync

**Sandbox**
- create_pod	
- sync_ready: Pod ready → set Running/Ready
- pause
- resume
- upgrade
- inplace_update
- terminating: delete/finalizer

**SandboxClaim 控制器(管批量 claim)**
- claim_batch
- completed: set status as completed
- expired	TTL 到期处理

**SandboxUpdateOps** 控制器(管批量升级已 claim 的)
⚠️ 这个现在零指标,内部分支我没细读,埋点时要先确认。大致:
- update_batch	推进一批更新
- rolling / partition	按策略进度
- completed	完成



### k8s side(sandbox level)

**current**: 
rest_client_request_duration_seconds{verb, host}   # 每次 apiserver 调用多久（客户端视角）
rest_client_requests_total{verb, code, host}        # 调用次数，按 verb + HTTP 状态码
request_duration:从发请求 → 收到响应,= 一次 apiserver 调用的耗时(含网络 + apiserver 处理 + etcd 写 + 返回)。
{verb}:GET/POST/PUT/PATCH/DELETE。
{code}:200/409(冲突)/429(APF 限流)…—— 所以锁冲突、限流能从这看到

