# Node watchdog

Replaces a worker node that stays NotReady/Unknown for more than 3 minutes, so Redis, Kafka and Jenkins reschedule
without manual work. EC2 Auto Scaling only checks the hypervisor, and Kubernetes never force-moves StatefulSet pods
off an unreachable node, so a hung kubelet used to leave the exchange down until a human terminated the instance.

## How it works
Runs every minute as a CronJob in `kube-system`:

1. Find a node that has been NotReady/Unknown for more than 180 s.
2. Ask AWS for the instance state.
   - `running` / `pending` / `stopping`: terminate the instance. Managed node group: ASG terminate with replacement.
     Karpenter or standalone node: `ec2 terminate-instances` directly (Karpenter then removes the NodeClaim itself).
     A NodeClaim delete is deliberately not used: Karpenter would first try to drain a node whose kubelet is dead and
     wait for volume detachment, which can hang.
   - `stopped` / `shutting-down` / `terminated` (or no longer found): add the
     `node.kubernetes.io/out-of-service` taint (pods are force-deleted, volumes force-detached). A later run deletes
     the Node object once the taint is present.
3. Separately, remove orphaned VolumeAttachments (their node is gone) so replacement pods are not blocked by
   "volume already attached".

## Safety
- One node per run; aborts (job fails) if more than `MAX_UNHEALTHY` (default 1) nodes are unhealthy, for example in
  an AZ outage or control-plane problem. Alert on a failed `node-watchdog` Job, because that exit code is the only signal.
- Never taints a node whose instance is still running.
- An AWS or Kubernetes call that fails unexpectedly makes the script skip or abort. It is never read as "instance gone".
- An orphaned VolumeAttachment is only deleted when AWS shows the EBS volume is not attached to a live instance.
  If the node list comes back empty, no VolumeAttachment is touched.
- The CronJob prefers (does not require) a non-Karpenter node, so it is unlikely to run on a node it must replace.

## IAM (terraform `node_watchdog.tf`)
The IRSA role needs:
- `ec2:DescribeInstances`, `ec2:DescribeVolumes`
- `ec2:TerminateInstances` (scope with a condition on your cluster tag)
- `autoscaling:TerminateInstanceInAutoScalingGroup` (scope to your node group ASGs)

## Deploy
1. `terraform apply` the `eks` module (adds `node_watchdog.tf`) and note `node_watchdog_role_arn`.
2. In `nodewatchdog.yaml`, put that ARN in the ServiceAccount annotation and set `AWS_REGION` to your cluster's region.
3. `kubectl apply -f nodewatchdog.yaml`
4. It ships with `DRY_RUN=true`. Read the logs for a day: `kubectl logs -n kube-system -l job-name`
   (only the latest failed job is kept; successful jobs are deleted). Then set `DRY_RUN` to `"false"` and re-apply.

## Test
Cordon is not enough. Test both node types separately, on a non-critical node:
- Managed node group node: stop the kubelet (`systemctl stop kubelet` via SSM).
- Karpenter node: do the same, and confirm the instance is terminated and the NodeClaim disappears.

Expected: NotReady -> 3 min -> instance terminated -> out-of-service taint -> Node deleted -> pods Running on a
replacement node.

## Limitations
- Detection granularity is one minute, so total recovery is roughly 4 to 5 minutes.
- Two or more simultaneously dead nodes are not replaced automatically (by design).
- The orphan cleanup needs the volume to be an EBS CSI volume (it resolves the PV's `volumeHandle`); other volume
  types are skipped.
