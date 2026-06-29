# OKA_KWOK_Loadtest

## Overview
开环ramp,100k
#### goal
#### method
#### output

## Global Sandboxset Test
#### method
#### metrics
- speed of creating = rate(`sandboxset_sandboxes_created_total{namespace,name}`)
- *speed of available/repleshment* = rate(`sandboxset_sandboxes_available_total{namespace, name}`)
- speed of delete = `sandboxset_sandboxes_deleted_total`(删除/缩容)
- rate of failure: `created_total{result="failure"}` / `created_total`
- 缺口1: `spec.Replicas` - available(gauge)

## E2E Test(e2b/cr)
### tested functions

| Method | Tested | Rationale |
|---|---|---|
| `ClaimSandbox()` | Y | Core op and the main source of load in real use; exercises all layers (manager + apiserver + controllers).|
| `PauseSandbox()`| Y | Core lifecycle op; exercises all layers.|
| `ResumeSandbox()`/`ConnectSandbox()` | Y | Core lifecycle op; exercises all layers.|
| `DeleteSandbox()`| Y |`Kill` is non-blocking, giving a clean manager-side capacity signal.|
| SandboxSet `spec.template` → UpdateRevision | Y |Rolling update of the pool (idle members, recreate); a main control-plane load source.|
| SandboxUpdateOps CR | Y | In-place update of claimed sandboxes; a main control-plane load source.|
| Batch claim/SandboxClaim CR | Y | CR claim path; bypasses the manager's claim-slot limit.|
| `CreateCheckpoint()`| N | Heavy work (memory/fs snapshot) runs in the real pod.|
| `CloneSandbox()` | N | Restore + re-init runs in the real pod (stubbed on Kwok); control-plane part ≈ Claim.|
| `SetTimeout()` | N | Cheap write on the same path as Pause's write.|
| `ListSandboxes()` | N | Not a main load source; a correctness concern, not a capacity one.|
| `GetClaimedSandbox()` | N | Single-object cache hit, negligible. |
| List snapshots / templates / API keys | N | Admin/read endpoints, not hot-path load. |


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
**metrics**: 
- p50, p90, p95, p99
- tps
- success count
- error count(409 conflict / 429 ratelimit / timeout / 500)

#### CR

- rolling update sandbox in sandboxset(change `SandboxSet spec.template` → UpdateRevision)  
  - metrics:
      - total time
      - rate(*sandboxset_sandboxes_updated_total{namespace,name}*) 

- rolling update claimed sandbox(`SandboxUpdateOps` cr), limited by    
   - metrics:
      - total time
      - speed(rate(`sandbox_inplace_update_duration_seconds_count`)) 



## Sandbox-manager test

### method

- T_manager(claim)  = Total − WaitReady − InitRuntime − CSIMount − SecurityToken = ΣPickAndLock(Total − WaitReady) + Σwait
    - `wait`: time waiting for claim worker slot
    - `PickAndLock`: pick an available sandbox and change pod/sandbox cr
    - WaitReady: wait pod/sandbox becoming ready
    - InitRuntime/CSIMount/SecurityToken: need running pod, all is 0 with kwok
- T_manager(pause)  = `sandbox_pause_duration`  − `sandbox_pause_wait` = `refresh` + `retryUpdate` + `InplaceRefresh`
  - `refresh`: read status and make sure pause can be done
  - `retryUpdate`: write `Spec.Paused=true`
  - `sandbox_pause_wait`: waiting controller pause sandbox
  - `InplaceRefresh`: read status again and check success
- T_manager(resume) = sandbox_resume_duration − sandbox_resume_wait = refresh + retryUpdate + InplaceRefresh
  - `refresh`: read status and make sure pause can be done
  - `retryUpdate`: write `Spec.Paused=true`
  - `sandbox_pause_wait`: waiting controller pause sandbox
  - `InplaceRefresh`: read status again and check success
- T_manager(delete) = sandbox_delete_duration (kill does not wait)


**metrics**: 
- p50, p90, p95, p99
- rate(rest_client_request_duration_seconds_sum{verb=~"GET|PUT|PATCH|POST|DELETE"}[5m])/ rate(段_sum[5m])
- process_cpu
- go_goroutines
- heap
- rest_client_rate_limiter_duration(客户端 QPS/Burst 限流)


## Sandbox-controller test

### controllers
- SandboxSet
- Sandbox
- SandboxClaim
- SandboxUpdateOps

### method

### metrics
#### work queue
- workqueue depth
- wait time(): rate(`workqueue_queue_duration_seconds_sum{name="sandboxset"}`[5m])/rate(`workqueue_queue_duration_seconds_count{name="sandboxset"}`[5m])
  

#### reconcile   
- p50,p90,p95,p99  
  rate(controller_runtime_reconcile_time_seconds_sum{controller})/rate(..._count{controller})
- api  
  apiserver 占比 ≈ rate(rest_client_request_duration_seconds_sum) / rate(reconcile_time_sum)

#### resource usage
- rate(process_cpu_seconds_total) —— 关键,区分 compute-bound vs 等队列/apiserver
- process_resident_memory_bytes /
- go_memstats_heap_* —— 100k 下 informer cache footprint
- go_goroutines、
- go_gc_duration_seconds
- (active_workers vs MaxConcurrentReconciles —— 已有)



### k8s side(sandbox level)

### method

### metrics
- `rest_client_rate_limiter_duration_seconds`: manager/controller's QPS/Burst ratelimitter
- `rest_client_request_duration_seconds{verb,host}`, rtt of rest_client call
- rest_client_requests_total{verb,code,host} 409 = 锁冲突,429 = APF 限流
- apiserver_request_duration_seconds
- apiserver_flowcontrol_*(rejected / waiting)—— APF 拒绝/排队,即 429 的根源
- etcd_disk_wal_fsync_duration_seconds —— WAL 落盘延迟(写的根瓶颈)
- etcd_request_duration_seconds —— etcd 操作延迟
- etcd_db_total_size —
- apiserver:CPU + 内存(apiserver 进程/容器)
- etcd:CPU + 内存,但更要紧的是磁盘——etcd 通常先 fsync/磁盘瓶颈,后 CPU。所以这层 etcd_disk_wal_fsync_duration + 磁盘 I/O 的优先级高于 CPU(已有 fsync)






