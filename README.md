# agent_loadtest

100threads*1000sandbox

## Tested API
How to decide start and end?
### Sandboxset(template)
state: creating/available    
**metric**: speed of claim vs speed of creation or replenishment        
**method**: 
- moniter changes of sandbox's number
- or record time of every sandbox statu change, calulate 

### Claim Sandbox
#### Claiming Sandboxes via E2B SDK
 
**metric**: latency of from sandbox claiming via E2B SDK to sandbox become running 
**baseline** Creating Sandbox Directly from Template  
**method**:  
```
start = time.time()

#claim sandbox and wait sandbox ready
with Sandbox.create(template="demo") as sbx:
    end = time.time()
    latency = end - start
```

#### Claiming Sandboxes via SandboxClaim
**metric**: latency of from sandbox claiming via E2B SDK to sandbox become running  
**baseline** Creating Sandbox Directly from Template  
**method**: 
```
start = time.time()

# submit SandboxClaim
kubectl.apply("sandboxclaim.yaml")

# Watch SandboxClaim statu
# wait Phase become Completed
watch SandboxClaim until phase == "Completed"

end = time.time()
latency = end - start
```

#### Batch Claiming Sandboxes
**metric**: latency of sandbox claiming via Batch Claiming(single and total latency)   
**baseline**: Creating Sandbox Directly from Template * replica
**method**: 
```
start = time.time()
apply_yaml("sandboxclaim.yaml")

seen = set()
running_times = []

for event in w.stream(...):
    obj = event["object"]
    name = obj["metadata"]["name"]
    phase = obj.get("status", {}).get("phase")

    if phase == "Running" and name not in seen:
        seen.add(name)
        running_times.append(time.time() - start)

    if len(running_times) >= replicas:
        end = time.time()
        latency = end - start
        w.stop()
        break

state_changing_per_second = Counter(running_times)

```
for multi-round test, dynamically generate yaml file every time

```
# "label:app" is added as a label on the associated Pod (key: "app")
Sandbox.create(template="demo", metadata={"label:app": "my-app", "label:env": "production"})

# You can mix labels and annotations
Sandbox.create(template="demo", metadata={"label:app": "my-app", "userId": "alice"})
```

```
Sandbox.create(template="demo", request_timeout=60.0, metadata={
    "e2b.agents.kruise.io/claim-timeout-seconds": "60"
})
```

#### Creating Sandbox Directly from Template(baseline? guarentee same config,use snapshot?)
```
start = time.time()
with Sandbox.create(template="demo", metadata={
    "e2b.agents.kruise.io/create-on-no-stock": "true"
}) as sbx:
    end = time.time()
    latency = end - start
```

### pause/resume
#### pause
**metric**: latency from func call to sandbox's status become paused  
**baseline**: no

#### resume
**metric**: latency from func call to sandbox's status become running  
**baseline**: Creating Sandbox Directly from Template

**method**:
```
# pause latency
start = time.time()
sbx.pause()

watcher = watch.Watch()
for event in watcher.stream(...):
    phase = event["object"].get("status", {}).get("phase")
    
    if phase == "Paused":   
        pause_latency = time.time() - start
        watcher.stop()
        break

# resume latency
start = time.time()
Sandbox.connect(sandbox_id)

for event in watcher.stream(...):
    phase = event["object"].get("status", {}).get("phase")
    
    if phase == "Running":    
        resume_latency = time.time() - start
        watcher.stop()
        break
```


### Image Replacement/resource adjustment
#### Upgrade Pre-warmed Pool Sandboxes (SandboxSet)
**metric**: speed of updating
**baseline**: 
**method**: watch version changing

#### Upgrade Claimed Sandboxes (SandboxUpdateOps)
