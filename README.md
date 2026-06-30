# OKA_KWOK_Loadtest

## Overview
#### goal  
Use **Kwok + kind** to simulate a large-scale cluster and evaluate OpenKruise
Agents at **100k-sandbox** scale: measure its performance under load, **find the system
bottleneck** — which of the three layers gives out first: the **sandbox-manager**
(synchronous E2B API), the **controllers** (asynchronous reconcile), or **k8s**
(apiserver + etcd) — and **detect performance regressions** against a baseline.


#### method  
Stand up a **100k-scale SandboxSet** as the pre-warmed pool, and drive
**sandbox churn at the same 100k scale** via an **open-loop ramp**.

## Global Sandboxset Test
#### method
#### metrics(general)
- speed of creating = rate(`sandboxset_sandboxes_created_total{namespace,name}`)
- *speed of available/replenishment* = rate(`sandboxset_sandboxes_available_total{namespace, name}`)
- gap: `spec.Replicas` - available(gauge)
- `sandboxset_sandboxes_claimed_total{namespace,name}`
- `sandboxset_updated_replicas`(gauge)
- rate of failure: `created_total{result="failure"}` / `created_total`


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
| List snapshots / templates / API keys | N | Admin/read endpoints, not hot-path load. |
| `GetClaimedSandbox()` | N | Single-object cache hit, negligible. |


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
- tps
- p50, p90, p95, p99
- success count
- error count(409 conflict / 429 ratelimit / timeout / 500)

#### CR

- batch claim via cr
  - metrics:
    - *contention*(core):retry of claim retry / 409 lock conflict  
    - total time: status.CompletionTime − creationTimestamp
    - process: status.UpdatedReplicas/Replicas
    - failure rate: failure count / batch size

- rolling update sandbox in sandboxset(change `SandboxSet spec.template` → UpdateRevision)  
  - metrics:
      - total time
      - speed:rate(*sandboxset_sandboxes_updated_total{namespace,name}*) 

- rolling update claimed sandbox(`SandboxUpdateOps` cr), limited by    
   - metrics:
      - total time
      - speed: rate(`sandbox_inplace_update_duration_seconds_count`))



## Sandbox-manager test

### method

- T_manager(claim)  = Total − WaitReady − InitRuntime − CSIMount − SecurityToken = ΣPickAndLock(Total − WaitReady) + Σwait
    - `wait`: time waiting for claim worker slot
    - `PickAndLock`: pick an available sandbox and change pod/sandbox cr
    - WaitReady: wait pod/sandbox becoming ready
    - InitRuntime(no sidecar)/CSIMount(no running pod)/SecurityToken(no running pod): need running pod, all is 0 with kwok
- T_manager(pause)  = `sandbox_pause_duration`  − `sandbox_pause_wait` = `refresh` + `retryUpdate` + `InplaceRefresh`
  - `refresh`: read status and make sure pause can be done
  - `retryUpdate`: write `Spec.Paused=true`
  - `sandbox_pause_wait`: waiting controller pause sandbox
  - `InplaceRefresh`: read status again and check success
- T_manager(resume) = sandbox_resume_duration − sandbox_resume_wait = refresh + retryUpdate + InplaceRefresh
  - `refresh`: read status and make sure resume can be done
  - `retryUpdate`: write `Spec.Paused=false`
  - `sandbox_pause_wait`: waiting controller resume sandbox
  - `InplaceRefresh`: read status again and check success
- T_manager(delete) = sandbox_delete_duration (kill does not wait)


**metrics**: 
- p50, p90, p95, p99
- rate(rest_client_request_duration_seconds_sum{verb="GET|PUT"}[5m])/rate(rest_client_request_duration_seconds_count{verb="GET|PUT"}[5m])
- process_cpu: cpu usage
- go_goroutines: goroutine usage
- heap: memory usage
- rest_client_rate_limiter_duration(client QPS/Burst ratelimitter)


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
- wait time(): rate(`workqueue_queue_duration_seconds_sum{name=controller}`[5m])/rate(`workqueue_queue_duration_seconds_count{name=controller}`[5m])
  

#### reconcile   
- p50,p90,p95,p99  
  rate(controller_runtime_reconcile_time_seconds_sum{controller})/rate(controller_runtime_reconcile_time_seconds_count{controller})
- apiserver  
  apiserver 占比 ≈ rate(rest_client_request_duration_seconds_sum) / rate(reconcile_time_sum)

#### resource usage（all controller）
- rate(process_cpu_seconds_total)
- process_resident_memory_bytes
- go_goroutines
- go_gc_duration_seconds
- active_workers(MaxConcurrentReconciles): current working reconcile worker



### k8s side(sandbox level)

### method

### metrics
- `rest_client_rate_limiter_duration_seconds`: manager/controller's QPS/Burst ratelimitter
- `rest_client_request_duration_seconds{verb,host}`, rtt of rest_client call
- rest_client_requests_total{verb,code,host} （409, 429(APF ratelimiter)）
- apiserver_request_duration_seconds
- apiserver_flowcontrol_*(rejected / waiting)—— APF ratelimiter count
- etcd_disk_wal_fsync_duration_seconds(core) —— time of fsync WAL
- etcd_request_duration_seconds —— etcd latency
- etcd_db_total_size 
- apiserver:CPU + memory
- etcd: CPU + memory





