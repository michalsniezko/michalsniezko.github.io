---
layout: default
title: SQS-Based Autoscaling
parent: CloudWatch, Autoscaling, Lambda Patterns
nav_order: 1
---

## SQS-Based Autoscaling

**Metric:** `ApproximateNumberOfMessagesVisible` — the number of messages in the queue waiting to be processed. More useful than CPU for worker-based architectures because a queue can be deep while workers sit idle waiting on I/O (low CPU, high backlog).

CPU-based scaling misses the actual problem: unprocessed work. A worker doing HTTP calls to a slow upstream might use 5% CPU while 50,000 messages pile up. SQS depth tells you how much work exists, not how hard the machines are working.

### Custom Metric: Backlog Per Instance

Raw queue depth doesn't account for fleet size. 10,000 messages with 2 workers is a problem; 10,000 messages with 100 workers is fine. The useful metric is **backlog per instance**.

```
BacklogPerInstance = ApproximateNumberOfMessagesVisible / RunningTaskCount
```

### Terraform: ECS Autoscaling on Queue Depth

```hcl
resource "aws_appautoscaling_target" "workers" {
  max_capacity       = 20
  min_capacity       = 2
  resource_id        = "service/${aws_ecs_cluster.main.name}/${aws_ecs_service.worker.name}"
  scalable_dimension = "ecs:service:DesiredCount"
  service_namespace  = "ecs"
}

resource "aws_appautoscaling_policy" "scale_on_queue" {
  name               = "worker-queue-depth-scaling"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.workers.resource_id
  scalable_dimension = aws_appautoscaling_target.workers.scalable_dimension
  service_namespace  = aws_appautoscaling_target.workers.service_namespace

  target_tracking_scaling_policy_configuration {
    target_value = 100  # aim for ~100 messages per worker

    customized_metric_specification {
      metric_name = "BacklogPerInstance"
      namespace   = "Custom/WorkerScaling"
      statistic   = "Average"
      unit        = "Count"
    }

    scale_in_cooldown  = 300  # wait 5 min before scaling down
    scale_out_cooldown = 60   # scale up quickly
  }
}
```

### Publishing the Custom Metric (Lambda or Cron)

```bash
#!/bin/bash
QUEUE_URL="https://sqs.eu-west-1.amazonaws.com/123456789012/order-processing"
CLUSTER="production"
SERVICE="order-worker"

MSG_COUNT=$(aws sqs get-queue-attributes \
    --queue-url "$QUEUE_URL" \
    --attribute-names ApproximateNumberOfMessagesVisible \
    --query 'Attributes.ApproximateNumberOfMessagesVisible' \
    --output text)

TASK_COUNT=$(aws ecs describe-services \
    --cluster "$CLUSTER" \
    --services "$SERVICE" \
    --query 'services[0].runningCount' \
    --output text)

# Avoid division by zero
if [ "$TASK_COUNT" -eq 0 ]; then
    BACKLOG="$MSG_COUNT"
else
    BACKLOG=$((MSG_COUNT / TASK_COUNT))
fi

aws cloudwatch put-metric-data \
    --namespace "Custom/WorkerScaling" \
    --metric-name "BacklogPerInstance" \
    --value "$BACKLOG" \
    --unit Count
```

> **Cost/Performance Note:** Set `scale_in_cooldown` significantly higher than `scale_out_cooldown`. Scaling out is cheap (handle the spike). Scaling in too fast causes **flapping**: traffic spike → scale to 20 → spike subsides → scale to 2 → next batch arrives → scale to 20. A 5-minute scale-in cooldown lets transient dips pass without thrashing. Also: ECS tasks take 30–90 seconds to become healthy — factor this startup lag into your target value.
