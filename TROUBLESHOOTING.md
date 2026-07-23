# Troubleshooting

## 1. Misconfigured alarm threshold (false alarm)

**Setup:** `cloudwatch-monitoring-high-memory` normally watches `mem_used_percent` with an 80% threshold, 2 evaluation periods of 300s (i.e. needs 10 min of sustained high memory before firing).

**Break:** Dropped the threshold to 1% and tightened the evaluation window to 1x60s, to force a false alarm.

**Diagnose:**
1. Confirmed the alarm flipped to `ALARM` state:
   ```
   aws cloudwatch describe-alarm-history --alarm-name cloudwatch-monitoring-high-memory
   ```
   Screenshot: `screenshots/04-break1-alarm-history.png`

2. Pulled the actual metric data for the same window to check whether the instance was really under memory pressure:
   ```
   aws cloudwatch get-metric-statistics --namespace CWAgent --metric-name mem_used_percent \
     --dimensions Name=InstanceId,Value=i-0c87cc8e8238ae7bc --start-time <window-start> \
     --end-time <window-end> --period 300 --statistics Average
   ```
   Result: memory usage was steady around 17-18% the whole time. Screenshot: `screenshots/05-break1-actual-metrics.png`

**Root cause:** The alarm itself was misconfigured (threshold set far below any realistic value), not an actual memory problem. The instance was healthy; the alarm config was the bug.

**Fix:** Reverted the alarm back to its production settings:
```
aws cloudwatch put-metric-alarm --alarm-name cloudwatch-monitoring-high-memory \
  --metric-name mem_used_percent --namespace CWAgent --statistic Average \
  --period 300 --evaluation-periods 2 --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=InstanceId,Value=i-0c87cc8e8238ae7bc \
  --alarm-actions arn:aws:sns:us-east-1:889168907206:cloudwatch-monitoring-alerts
```

**Note on recovery lag:** Right after reverting, the alarm stayed in `ALARM` state — this is expected, not a bug. Alarms only re-evaluate on their configured schedule, so it needs 2 full new periods (~10 min) of fresh data under the reverted threshold before it can flip back to `OK`. Confirmed settled the next day:
```
aws cloudwatch describe-alarms --alarm-names cloudwatch-monitoring-high-memory
```
`StateValue: OK`, `StateReason`: memory at ~15.5%, well under the 80% threshold.

**Lesson:** When an alarm fires, check the raw metric data before assuming the underlying resource is unhealthy — the alarm configuration itself is part of the system that can fail. Also, alarm state changes aren't instantaneous after a config fix; budget for the evaluation window before declaring it resolved.

## 2. Stopped CloudWatch agent (metrics stop flowing)

**Break:** Stopped the agent via SSM Run Command:
```
aws ssm send-command --instance-ids i-0c87cc8e8238ae7bc \
  --document-name "AmazonCloudWatch-ManageAgent" --parameters '{"action":["stop"],"mode":["ec2"]}'
```

**Diagnose:**
1. Checked the metric data directly — confirmed no new datapoints after the stop:
   ```
   aws cloudwatch get-metric-statistics --namespace CWAgent --metric-name mem_used_percent \
     --dimensions Name=InstanceId,Value=i-0c87cc8e8238ae7bc \
     --start-time <15 min ago> --end-time <now> --period 300 --statistics Average
   ```
   Result: `"Datapoints": []` — nothing flowing.

2. Checked the agent process itself on the instance (not just the alarm/metric side) via SSM Run Command:
   ```
   aws ssm send-command --instance-ids i-0c87cc8e8238ae7bc \
     --document-name "AWS-RunShellScript" \
     --parameters '{"commands":["sudo systemctl status amazon-cloudwatch-agent"]}'
   ```
   Output confirmed: `Active: inactive (dead) since ... UTC`, clean exit (`code=exited, status=0/SUCCESS`). Note: SSM reported this command as `Status: Failed` with exit code 3 — that's expected and not a real failure, since `systemctl status` always returns exit code 3 for an inactive service; the actual evidence is in the stdout content, not the exit status.

**Root cause:** Agent process stopped intentionally (simulating a crash/manual stop in the real world) — clean shutdown, not a crash or permission issue.

**Fix:** Restarted the agent the same way, with `"action":["start"]`, and confirmed metrics resumed flowing on the next check.

**Lesson:** Empty metrics can come from two different places — the alarm/metric side (nothing to evaluate) or the agent/instance side (nothing being published). Diagnosing which requires checking the source (agent process + logs), not just the CloudWatch console. See callout above for the harder-to-spot permissions variant of this same symptom.

> **Callout — permissions vs. process failure:** this break stops the agent process entirely, which is visible via `systemctl status amazon-cloudwatch-agent` (shows inactive/dead). In a real environment, a more deceptive variant of this same symptom is the instance role losing `cloudwatch:PutMetricData` permission (e.g. `CloudWatchAgentServerPolicy` detached, an SCP blocking the action, or a permission boundary change) — the agent process stays "active" and `systemctl status` looks healthy, but every publish call gets silently rejected. That variant can't be diagnosed from process/service status at all; you have to check the agent's own log for `AccessDenied`/`UnauthorizedOperation` errors:
> ```
> sudo tail -50 /opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log
> ```
> Lesson: "metrics stopped" has at least two distinct root causes (process down vs. permission denied) that look identical from CloudWatch's side and require different diagnostic paths — service status for the former, agent logs for the latter.
