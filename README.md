# agent_loadtest

## Tested API
### Sandboxset(template)
state: creating/available  
claim and replenish  
**metric**: latency of sandbox creation    
**method**: watch state changes(time from creating start to available)  

### Claim Sandbox
#### Claiming Sandboxes via E2B SDK
```
with Sandbox.create(template="demo") as sbx:
    print(sbx.get_info())
```
template: The name of the `SandboxSet`  
**metric**: latency of sandbox creation    
**method**: calculate time from SDK to sandbox become `completed`)   

#### Claiming Sandboxes via SandboxClaim
**metric**: latency of sandbox creation  
**method**: calculate time from cli execution to sandbox become `completed`) 

#### Batch Claiming Sandboxes
`claimTimeout`: Specifies the timeout for the claim. Maximum time to attempt working.   
`ttlAfterCompleted`: Specifies the TTL time after the claim completes. After the claim task completes and the TTL time passes, the SandboxClaim resource will be deleted (the claimed sandboxes will not be deleted).
```
Sandbox.create(template="demo", request_timeout=60.0, metadata={
    "e2b.agents.kruise.io/claim-timeout-seconds": "60"
})
```

#### Creating Sandbox Directly from Template(baseline?)
```
Sandbox.create(template="demo", metadata={
    "e2b.agents.kruise.io/create-on-no-stock": "true"
})
```




```
# "label:app" is added as a label on the associated Pod (key: "app")
Sandbox.create(template="demo", metadata={"label:app": "my-app", "label:env": "production"})

# You can mix labels and annotations
Sandbox.create(template="demo", metadata={"label:app": "my-app", "userId": "alice"})
```

#### Image Replacement/resource adjustment
replace container/resource adjustment image when claiming a sandbox
claiming, in-place upgrade（pull image, restart,slow）
creating, a new Sandbox will be created

### pause/resume
1. E2B SDK（代码里调用）
   sbx.beta_pause()  # 暂停
   sbx.connect()     # 恢复
2. K8s CRD（kubectl 命令）
   kubectl edit sandbox my-sandbox
   → 修改 spec.paused = true   # 暂停
   → 修改 spec.paused = false  # 恢复
