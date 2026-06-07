# Runbooks

## Alert: MinecraftServerDown

**What it means:** mc-monitor cant reach the Minecraft server on port 25565. Players are getting connection refused. This is player-visible.

**First check:**
kubectl get pods -n minecraft

**If pod is Running but server is still down:**
kubectl logs -n minecraft -l app=minecraft --tail=50

Look for Java exceptions or OOM errors.

**If pod is in ErrImagePull or ImagePullBackOff:**
kubectl describe pod -n minecraft -l app=minecraft

Check the Events section. Bad image tag or ECR auth issue.

**If pod is CrashLoopBackOff:**
kubectl logs -n minecraft -l app=minecraft --previous --tail=50

**Recovery:**
- Bad deploy: kubectl rollout undo deployment/minecraft -n minecraft
- OOM kill: increase memory limit in deployment.yaml and reapply
- Healthy pod but no response: kubectl delete pod -n minecraft -l app=minecraft

**Confirm recovery:** Dashboard Server Status panel shows UP.

---

## Alert: MinecraftPodCrashLooping

**What it means:** The Minecraft container has restarted more than 3 times in 15 minutes. The server is unstable and players are getting dropped repeatedly.

**First check:**
kubectl get pods -n minecraft

Note the RESTARTS count and AGE.

**Get crash reason:**
kubectl logs -n minecraft -l app=minecraft --previous --tail=100

Common causes:
- Out of memory: look for java.lang.OutOfMemoryError
- Bad config: look for startup errors referencing server.properties
- Corrupt world: look for java.io.IOException on world files

**Check recent events:**
kubectl get events -n minecraft --sort-by=.lastTimestamp | tail -20

**Recovery:**
- OOM: edit configmap MEMORY value down, or increase memory limit in deployment.yaml
- Bad config: kubectl edit configmap minecraft-config -n minecraft, fix the value, delete the pod
- Corrupt world: restore from S3 backup

**Confirm recovery:** RESTARTS count stops climbing, pod stays Running.

---

## Alert: NodeMemoryPressure

**What it means:** The node has been above 85% memory usage for 5 minutes. No pods have been killed yet but the node is close to the limit. If it hits 100% the kernel will start OOM-killing processes.

**First check:**
kubectl top pods -A --sort-by=memory

Find the biggest memory consumers.

**Check node pressure conditions:**
kubectl describe node ip-10-0-1-75 | grep -A5 Conditions

Look for MemoryPressure: True.

**Quick wins:**
- Minecraft is usually the biggest consumer. Check if MEMORY in the configmap is set too high.
- Check if Prometheus is eating memory: kubectl top pods -n monitoring

**Recovery:**
- Reduce Minecraft memory: edit configmap, change MEMORY from 2G to 1500m, delete the minecraft pod
- If monitoring is the culprit: values.yaml already caps Prometheus at 700Mi and Grafana at 200Mi
- If node is already OOM killing: reboot the instance from AWS console as last resort

**Confirm recovery:** Node Memory percent panel drops below 80%.
