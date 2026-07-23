# LinkedIn Post — CloudWatch Monitoring & Alarms

---

I set up full observability on an EC2 instance this week — then broke it twice on purpose to see if I'd actually catch it. 🧵

**What I built:**
- Installed the CloudWatch agent via SSM Run Command (no manual SSH)
- Custom metrics the default EC2 dashboard doesn't give you for free: memory, disk, and swap usage
- Two alarms wired to SNS for email alerts
- A live dashboard with threshold annotation lines so I can see at a glance how close I am to tripping an alarm

Agent config pushed via SSM Parameter Store, collecting under the `CWAgent` namespace:

```json
{
  "metrics": {
    "namespace": "CWAgent",
    "metrics_collected": {
      "mem": { "measurement": ["mem_used_percent"] },
      "disk": { "measurement": ["disk_used_percent"], "resources": ["/"] },
      "swap": { "measurement": ["swap_used_percent"] }
    }
  }
}
```

**Then I broke it. Twice.**

→ **Break 1 — the false alarm.** I dropped the memory alarm's threshold to 1%, and it fired immediately. Before assuming the instance was in trouble, I pulled the raw metric data directly — memory was steady at ~17%. The instance was healthy; the *alarm config* was the bug. Lesson: when something pages you, check the source data before you trust the alert.

→ **Break 2 — the silent stop.** I stopped the CloudWatch agent via SSM and watched metrics go flat. `systemctl status` showed a clean `inactive (dead)` — but that's the easy case. In a real environment, a permissions failure (say, the instance role losing `cloudwatch:PutMetricData`) produces the *exact same symptom* — metrics stop — except the agent process still shows "active." You can't tell those two apart from the dashboard. You have to go to the agent's own log and look for `AccessDenied` errors. Same failure, two completely different root causes, two different diagnostic paths.

**What this reinforced:** monitoring isn't done when the dashboard turns green. It's done when you've verified you can actually tell a real incident apart from a misconfigured alarm — and that you know where to look when the obvious check (service status) doesn't tell the whole story.

Full build + both break/debug writeups (with commands and screenshots) on GitHub 👇
https://github.com/GE224/cloudwatch-monitoring

#AWS #CloudWatch #Observability #CloudEngineering #DevOps #SRE #LearningInPublic
