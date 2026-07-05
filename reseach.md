e2b create.go `createSandboxWithClaim()`
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

api/v1alpha1/sandboxclaim_types.go:116


`SkipInitRuntime bool json:"skipInitRuntime,omitempty"`





