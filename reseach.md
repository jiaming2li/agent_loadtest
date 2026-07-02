sandbox ready后，InitRuntime / Token / CSIMount(7-9,连 envd)，路由设置(CR/proxy 状态)

Manager 调用(同步,在请求里)
1. claim — TryClaimSandbox(claim.go:220),当 opts.InitRuntime != nil:

何时:领到沙箱(lock + 等 Ready)之后、返回之前;
为什么:给刚领到的沙箱做首次运行时初始化(env、工作目录、access token 灌给 envd)。

2. resume / upgrade — initializer.Initialize(sandbox_initializer.go:119):

何时:sandbox 控制器处理 resume 或 upgrade 时(pod 恢复/替换后);
为什么:pod 被暂停(休眠)或升级(重建)过,运行时状态没了 → 回来时要重新 init;
就是这处卡住了 resume 的 Ready(我们之前查的):Initialize 失败 → Ready 不设 → resume 挂住。


InitRuntime 到底在干什么
InitRuntimeOptions(types.go:23)就这几个:
所以 POST /init 给 envd = manager 把「这次会话的环境变量 + access token」发给沙箱里的 envd,让 envd 应用上去。
池子里预热好的沙箱是通用的、空白的(env 没设、没绑定谁)。你 claim 到它之后,InitRuntime 就是把它"配置成你的":




 Sandbox Pod  (一个 IP,大家共享)
├── 容器①:用户主容器(跑代码的环境)
├── 容器②:agent-runtime (envd)   ← sidecar,独立容器
├── 容器③:csi-sidecar
└── ...


POST /sandboxes → ClaimSandbox → TryClaimSandbox:
  1. 等 claim 槽
  2. pick 一个 available 沙箱(cache 读)
  3. 打 claim 标记(owner/claimed)
  4. lock:Update 写 apiserver,占为己有       ← 到这就"领到"了
  5. 放槽
  6. (快路径:沙箱本就 Ready,几乎不等)
  7. InitRuntime:POST /init 给 envd → envd 在沙箱里搭好运行时(env、工作目录)
  8. SecurityToken:签发 + patch 记录 + 推给 envd(当凭证)
  9. CSIMount:经 envd 在 pod 里挂持久卷
  → 返回沙箱(带 pod IP、access token)


envd(agent-runtime)是沙箱 pod 模板(SandboxTemplate)里定义的 sidecar 容器，所以控制器从模板 mint 出来的每个沙箱 pod 都自带它

 Sandbox Pod  (一个 IP,大家共享)
├── 容器①:用户主容器(跑代码的环境)
├── 容器②:agent-runtime (envd)   ← sidecar,独立容器
├── 容器③:csi-sidecar
└── ...

网络(同一个 pod IP):所以外面用 pod IP 就能找到 envd;envd 和主容器之间可以走 localhost 互通;
(需要时)卷


主容器 Containers[0](用户的)	就是沙箱环境本身:用户选的镜像(如 code-interpreter)——OS、工具、库、文件系统都在这。代码/进程真正运行的地方
agent-runtime (envd) sidecar	API 守护进程:从外面(pod IP)接 run_code/files/commands,然后在那个共享环境里把它执行掉
它们共享卷(沙箱文件系统)(查到 sidecar 注入就是往主容器和 sidecar 挂共享 volume)——所以 envd 读写的文件、跑的东西,和主容器看到的是同一套环境/文件系统。


一个诚实的边界:envd 到底是"自己在 sidecar 里跑命令(靠共享文件系统)"还是"exec 进主容器跑",这个精确机制我没完全查实(查到的是共享 volume,没看到共享 PID 命名空间)。角色分工(主=环境、envd=API/执行前端)是确定的。要我去 cmd/agent-runtime/ 把 envd 具体怎么执行命令(在哪个容器上下文)查清楚吗?



Step 3  modifyPickedSandbox           改内存:清 ownerRef([claim.go:550]) + 打 claimed 标记([claim.go:555])  ← 在这清空


r.Delete(pod) 只是告诉 apiserver 要删,真正拆是 kubelet:


1. apiserver 给 pod 打 deletionTimestamp + 优雅期 → pod 进入 Terminating
2. 节点上的 kubelet 看到:
   → 跑 preStop hook(若有)
   → 给容器(主容器 + envd + csi sidecar)发 SIGTERM
   → 等 terminationGracePeriodSeconds(默认 30s)
   → 还没退就 SIGKILL
3. 容器停 → kubelet 清理:卸载卷、释放 pod IP/网络
4. kubelet 报告 pod 没了 → apiserver 删除 Pod 对象

CSI = Container Storage Interface,是 Kubernetes 挂载外部存储的标准插件接口(让各种存储后端——云盘、NFS、对象存储等——能挂进容器)。

csi-sidecar = 沙箱 pod 里一个专门负责挂载存储的辅助容器。

pod的condition=sandbox的condition（ready）

```
// syncSandboxStatusFromPod updates sandbox status from pod info and syncs the Ready condition
// with container startup failure detection.
func syncSandboxStatusFromPod(pod *corev1.Pod, newStatus *agentsv1alpha1.SandboxStatus) {
	newStatus.NodeName = pod.Spec.NodeName
	newStatus.SandboxIp = pod.Status.PodIP
	newStatus.PodInfo = agentsv1alpha1.PodInfo{
		PodIP:    pod.Status.PodIP,
		NodeName: pod.Spec.NodeName,
		PodUID:   pod.UID,
	}
	pCond := utils.GetPodCondition(&pod.Status, corev1.PodReady) //从 Pod 的 Status 里查找类型为 PodReady 的那个 condition,取出来赋值给 pCond。
	cond := utils.GetSandboxCondition(newStatus, string(agentsv1alpha1.SandboxConditionReady))
	if cond == nil {
		cond = &metav1.Condition{
			Type:               string(agentsv1alpha1.SandboxConditionReady),
			Status:             metav1.ConditionFalse,
			LastTransitionTime: metav1.Now(),
			Reason:             agentsv1alpha1.SandboxReadyReasonPodReady,
		}
	}
	if pCond != nil && string(pCond.Status) != string(cond.Status) {
		cond.Status = metav1.ConditionStatus(pCond.Status)
		cond.LastTransitionTime = pCond.LastTransitionTime
		cond.Reason = agentsv1alpha1.SandboxReadyReasonPodReady
		cond.Message = ""
	}
...

	utils.SetSandboxCondition(newStatus, *cond)
}
```
```
func (r *commonControl) EnsureSandboxRunning(ctx context.Context, args EnsureFuncArgs) (time.Duration, error) {
	pod, box, newStatus := args.Pod, args.Box, args.NewStatus
	// If the Pod does not exist, it must first be created.
	if pod == nil {
		if requeueAfter, shouldReturn := r.rateLimiter.getRateLimitDuration(ctx, pod, box); shouldReturn {
			return requeueAfter, nil
		}
		_, err := r.createPod(ctx, box, newStatus)
		return 0, err
	}

	// pod status running
	if pod.Status.Phase == corev1.PodRunning {
		newStatus.Phase = agentsv1alpha1.SandboxRunning
		syncSandboxStatusFromPod(pod, newStatus)
		return 0, nil
	}

	return 0, nil
}
```
