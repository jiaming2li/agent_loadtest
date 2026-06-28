# OKA_KWOK_Loadtest

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

| Function | Entry point | Tested | Layers exercised | Rationale |
|---|---|:---:|---|---|
| **Claim** | `POST /sandboxes` (pool) | ✅ | manager + apiserver + sandbox/sandboxset ctrl | Primary user op. Pick + lock + retry path; draining the pool also exercises refill. |
| **Pause** | `POST /sandboxes/{id}/pause` | ✅ | manager (thin) + sandbox ctrl | Core lifecycle. Manager write + controller convergence (`pauseTask.Wait`). |
| **Resume** | `POST /sandboxes/{id}/connect` | ✅ | manager (thin) + sandbox ctrl | Core lifecycle. New SDK resumes via `connect`; a real resume only when the sandbox is `Paused` (assert `201`). |
| **Delete** | `DELETE /sandboxes/{id}` | ✅ | manager (thin) + sandbox ctrl + sandboxset (scale) | Core lifecycle. `Kill` is non-blocking → `delete_duration` is already a clean manager segment. |
| **Rolling update** | SandboxSet `spec.template` → UpdateRevision | ✅ | sandboxset ctrl (rollout) | Distinct controller path none of the four touch. |
| **In-place update** | SandboxUpdateOps CR | ⚠️ | sandboxupdateops ctrl | Test, **but** this controller has **zero metrics** today — add batch-progress + convergence instrumentation first. |
| **Checkpoint / Snapshot** | `POST /sandboxes/{id}/snapshots` | ❌ | (real sidecar/node) | Snapshot = memory/fs dump of a **real** container. Kwok fakes pods → the heavy work cannot run; only CR bookkeeping remains, which apiserver/etcd capacity already covers. |
| **Clone** | `POST /sandboxes` (checkpoint id) | ❌ | (real sidecar/node) | Distinguishing cost is restore + `ReInitRuntime` + CSI remount — exactly what is stubbed on Kwok. Control-plane part ≈ Create. |
| **Set timeout** | `POST /sandboxes/{id}/timeout` | ❌ | manager (thin) + apiserver | Cheap write on the same path as Pause's write — no new coverage. |
| **Connect (already Running)** | `POST /sandboxes/{id}/connect` | ❌ | manager (thin) | When not paused it is an extend-only timeout write; covered as the no-op branch of Resume. |
| **List** | `GET /v2/sandboxes` | ❌ | manager (cache read) | No one lists 100k; a realistic list is owner-filtered + paginated → cheap. The manager's memory ceiling is already exercised by the 100k informer footprint. |
| **Describe** | `GET /sandboxes/{id}` | ❌ | manager (cache read) | Single-object cache hit, negligible. |
| **Batch claim** | SandboxClaim CR | ❌\* | sandboxclaim ctrl | Skip unless batch claiming is a real workload; manager Claim already covers claim semantics. (\*flip to ⚠️ if in scope.) |
| **List snapshots / templates / API keys** | `GET /snapshots`, `/templates`, `/api-keys` | ❌ | manager (read / admin) | Admin/read endpoints, not hot-path load. |


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

- rolling update(change `SandboxSet spec.template` → UpdateRevision)  
  - metrics:
      - total time
      - rate(*sandboxset_sandboxes_updated_total{namespace,name}*) 

- in-place update(`SandboxUpdateOps` cr)  
   - metrics:
      - total time
      - speed(rate(`sandbox_inplace_update_duration_seconds_count`)) 



## Sandbox-manager test

### method

```
T_manager(pause)  = sandbox_pause_duration  − sandbox_pause_wait = refresh + retryUpdate + InplaceRefresh
T_manager(resume) = sandbox_resume_duration − sandbox_resume_wait = refresh + retryUpdate + InplaceRefresh
T_manager(claim)  = Total − WaitReady − InitRuntime − CSIMount − SecurityToken = ΣPickAndLock(Total − WaitReady) + Σwait
T_manager(delete) = sandbox_delete_duration (kill does not wait)

```
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



### k8s side(sandbox level)

**current**: 
rest_client_request_duration_seconds{verb, host}   # 每次 apiserver 调用多久（客户端视角）
rest_client_requests_total{verb, code, host}        # 调用次数，按 verb + HTTP 状态码
request_duration:从发请求 → 收到响应,= 一次 apiserver 调用的耗时(含网络 + apiserver 处理 + etcd 写 + 返回)。
{verb}:GET/POST/PUT/PATCH/DELETE。
{code}:200/409(冲突)/429(APF 限流)…—— 所以锁冲突、限流能从这看到

