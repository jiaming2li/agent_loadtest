
### Pause

#### Manager side
`sandbox.go:311-364`

```
if err = pauseTask.Wait(time.Minute); err != nil {
		log.Error(err, "failed to wait sandbox pause")
		return err
	}
```
`pauseTask.Wait(time.Minute)`: 等待sandbox到达 paused 状态(SandboxPaused 条件=True / phase=Paused)（最多 1min）



#### Controller side   
`spec.Paused=true` 触发 reconcile:`common_control.go: EnsureSandboxPaused(), 191-247`


```
if rejected := r.checkpointControl.AssumePodCheckpointed(ctx, pod, box, newStatus, cond); rejected {
		return nil
	}

	err := client.IgnoreNotFound(r.Delete(ctx, pod, &client.DeleteOptions{GracePeriodSeconds: ptr.To(int64(5))}))
	if err != nil {
```
- r.Delete()
- 假 pod 要有真实的 containerStatuses(imageID),否则 AssumePodCheckpointed 可能记录不全。别在 kwok 上开 SandboxPauseCheckpoint(Alpha,默认关)


4. 下次 reconcile：pod == nil → Paused 条件 = True(reason=DeletePod)
     → phase = Paused，完成



### Resume

#### Manager side
`sandbox.go:373-461`

```
if err = resumeTask.Wait(resumeWaitMaxTimeout); err != nil {
		// A canceled request context surfaces here as "client rate limiter Wait
		// returned an error: context canceled" from client-go, which is misleading:
		// it is not a throttling failure but a propagation of ctx.Err(). Log it at
		// Info level so it is not mistaken for a server-side error in metrics or
		// alerting pipelines that watch klog ERROR.
		if ctxErr := ctx.Err(); ctxErr != nil {
			log.Info("stop waiting sandbox resume: request canceled by client (disconnected or client-side timeout)",
				"err", err, "ctxErr", ctxErr)
			return err
		}
		log.Error(err, "failed to wait sandbox resume")
		return err
	}
```
`resumeTask.Wait(resumeWaitMaxTimeout)`: 等待sandbox到达 running 状态(resume = 重建 pod + 等 ready + 重做 runtime/CSptrI 初始化)（最多 1min）



#### Controller side   
`spec.Paused=false` 触发 reconcile:`common_control.go: EnsureSandboxPaused(), 249-299`


```
if rejected := r.checkpointControl.AssumePodCheckpointed(ctx, pod, box, newStatus, cond); rejected {
		return nil
	}

	err := client.IgnoreNotFound(r.Delete(ctx, pod, &client.DeleteOptions{GracePeriodSeconds: ptr.To(int64(5))}))
	if err != nil {
```
- r.Delete()
- 假 pod 要有真实的 containerStatuses(imageID),否则 AssumePodCheckpointed 可能记录不全。别在 kwok 上开 SandboxPauseCheckpoint(Alpha,默认关)



### Claim


### Inplace-update
