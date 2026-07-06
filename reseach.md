#### e2b create.go `createSandboxWithClaim()`
```
if !request.Extensions.SkipInitRuntime {
		opts.InitRuntime = &config.InitRuntimeOptions{
			EnvVars:     request.EnvVars,
			AccessToken: accessToken,
		}
	}
```

pkg/sandbox-manager/infra/sandboxcr/claim.go:218-227

```
if opts.InitRuntime != nil {
    metrics.InitRuntime, err = runtime.InitRuntime(ctx, sbx.Sandbox, *opts.InitRuntime, ...)
    if err != nil {
        err = retriableError{...}   // ← 失败返回可重试错误(重试风暴的源头)
        return
    }
}
```
#### cr
api/v1alpha1/sandboxclaim_types.go:116


`SkipInitRuntime bool json:"skipInitRuntime,omitempty"`

### claim_with_update（修改对应spec,等待reconcile）
```
curl -X POST http://127.0.0.1:8080/sandboxes \
  -H "Host: api.localhost" -H "X-API-KEY: some-api-key" -H "Content-Type: application/json" \
  -d '{
    "templateID": "loadtest",
    "metadata": {
      "e2b.agents.kruise.io/skip-init-runtime": "true",
      "e2b.agents.kruise.io/image": "busybox:1.36",
      "e2b.agents.kruise.io/cpu-request": "200m",
      "e2b.agents.kruise.io/cpu-limit": "500m"
    }
  }'
```

```
create.go:
  parseExtensionImage (extensions.go:152)     ← 读 metadata["...image"]
  parseExtensionResources (extensions.go:165) ← 读 cpu-request/cpu-limit
  → request.Extensions.InplaceUpdate
create.go:116:
  if InplaceUpdate.Image != "" || Resources != nil {
    opts.InplaceUpdate = {Image, Resources}
  }
→ claim.go:174 pick 阶段 modifyPickedSandbox 应用 + 走 in-place 更新流程
```

`sandbox_controller.go`:245 : `err = r.getControl(args.Pod).EnsureSandboxUpdated(ctx, args)`

`common_control.go:105`: `EnsureSandboxUpdated()` 
common_inplace_update_handler.go:49 handleInPlaceUpdateCommon




