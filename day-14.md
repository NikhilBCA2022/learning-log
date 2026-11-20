## Day 14: Scaling, Rollouts, Failure Drills

### HPA (api-gateway)
- At rest: 2%/50% CPU, 1 replica.
- Under load: 336%/50%, scaled 1 -> 5 pods within about a minute, capped by maxReplicas=5.
- New pods start 0/1 and only receive traffic after the readiness probe passes.
- Scale-down: <fill in: time it took to return to 1 replica>
- Scale-up is fast to protect users. Scale-down waits about 5 minutes (stabilization window) so the HPA doesn't flap.

### Rolling update and rollback
- Tagged the same image as v2 in ECR, `kubectl set image` to v2, `rollout status` confirmed completion: <fill in: worked / what you saw>
- `kubectl rollout undo` returned the image to v1: <fill in: confirmed image tag>

### CrashLoopBackOff drill
- Patched user-service with `command: sh -c "echo boom; exit 1"`.
- New pod went Error -> CrashLoopBackOff, restarts climbing; old pod kept serving traffic, so the app stayed up.
- Diagnosis: `kubectl describe pod <name>` showed Last State Terminated, Exit Code 1.
  `kubectl logs <name> --previous` shows the crashed container's output.
- Gotcha: `-l app=...` matched both the old and new pod, so I had to name the crashing pod explicitly.
- Fixed with `kubectl rollout undo deploy/user-service`.

### Failure patterns learned
| Symptom | First command | Typical cause |
|---|---|---|
| Pending | `describe pod` (Events) | resources, taints, quota |
| ImagePullBackOff | `describe pod` (Events) | wrong tag, no ECR access |
| CrashLoopBackOff | `logs --previous`, `describe` (exit code) | app or command crashes on start |
