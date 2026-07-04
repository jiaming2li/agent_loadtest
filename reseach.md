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


SkipInitRuntime bool `json:"skipInitRuntime,omitempty"`


common_control.go:331 
```
if !claim.Spec.SkipInitRuntime {
		hasAgentRuntime := false
		// Check condition A: Runtimes field contains agent-runtime
		for _, rt := range sandboxSet.Spec.Runtimes {
			if rt.Name == agentsv1alpha1.RuntimeConfigForInjectAgentRuntime {
				hasAgentRuntime = true
				break
			}
		}
		// Check condition B: initContainer named "runtime"
		if !hasAgentRuntime {
			podTemplateSpec, err := utils.GetTemplateSpec(ctx, c.Client, sandboxSet.Namespace, &sandboxSet.Spec.EmbeddedSandboxTemplate)
			if err != nil {
				if sandboxSet.Spec.TemplateRef != nil {
					logger.Error(err, "failed to get sandbox template for checking agent runtime", "template", sandboxSet.Spec.TemplateRef.Name)
				} else {
					logger.Error(err, "failed to get sandbox template for checking agent runtime")
				}
				return opts, err
			}

			if podTemplateSpec != nil {
				for _, container := range podTemplateSpec.Spec.InitContainers {
					if container.Name == common.RuntimeInitContainerName {
						hasAgentRuntime = true
						break
					}
				}
			}
		}

		if hasAgentRuntime {
			opts.InitRuntime = &config.InitRuntimeOptions{
				EnvVars:     claim.Spec.EnvVars,
				AccessToken: config.NewDefaultAccessToken(),
			}
		} else {
			logger.Error(fmt.Errorf("agent-runtime not configured in SandboxSet"), "SkipInitRuntime is false but no agent-runtime found, skip InitRuntime",
				"sandboxSet", klog.KObj(sandboxSet), "claim", klog.KObj(claim))
		}
	}
```




