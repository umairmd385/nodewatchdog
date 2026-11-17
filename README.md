# Node watchdog

Replaces a worker node that stays NotReady/Unknown for more than 3 minutes, so Redis, Kafka and Jenkins reschedule
without manual work. EC2 Auto Scaling only checks the hypervisor, and Kubernetes never force-moves StatefulSet pods
off an unreachable node, so a hung kubelet used to leave the exchange down until a human terminated the instance.

Flow (runs every minute): find node NotReady > 180 s -> terminate the instance (Karpenter node: delete NodeClaim;
managed node group: ASG terminate with replacement) -> once AWS shows it shutting down, add the
`node.kubernetes.io/out-of-service` taint (volumes are force-detached, StatefulSet pods are force-deleted) and delete
the Node object.

Also removes orphaned VolumeAttachments (node already gone) so replacement pods are not blocked by "volume already attached".

Safety: one node per run; aborts if more than one node is unhealthy; never taints a node whose instance is still running.

## Deploy
1. `terraform apply` the `eks` module (adds `node_watchdog.tf`) and note `node_watchdog_role_arn`.
2. Put that ARN in `node-watchdog.yaml`, then `kubectl apply -f node-watchdog.yaml`.
3. It ships with `DRY_RUN=true`. Check `kubectl logs -n kube-system -l job-name` for a day, then set `DRY_RUN` to `"false"`.

## Test
Cordon is not enough; to rehearse, stop the kubelet on one non-critical node (`systemctl stop kubelet` via SSM) and
watch: NotReady -> 3 min -> instance replaced -> pods Running.
