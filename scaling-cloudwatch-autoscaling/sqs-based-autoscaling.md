---
layout: default
title: SQS-Based Autoscaling
parent: CloudWatch, Autoscaling, Lambda Patterns
nav_order: 1
---

## SQS-Based Autoscaling

**Metric:** `ApproximateNumberOfMessagesVisible` - the number of messages in the queue waiting to be processed. More useful than CPU for worker-based architectures because a queue can be deep while workers sit idle waiting on I/O (low CPU, high backlog).

CPU-based scaling misses the actual problem: unprocessed work. A worker doing HTTP calls to a slow upstream might use 5% CPU while 50,000 messages pile up. SQS depth tells you how much work exists, not how hard the machines are working.

### Custom Metric: Backlog Per Instance

Raw queue depth doesn't account for fleet size. 10,000 messages with 2 workers is a problem; 10,000 messages with 100 workers is fine. The useful metric is **backlog per instance**.

```
BacklogPerInstance = ApproximateNumberOfMessagesVisible / RunningTaskCount
```

### Terraform: ECS Autoscaling on Queue Depth

[`aws_appautoscaling_policy`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/appautoscaling_policy) with `TargetTrackingScaling` adjusts the ECS service's desired count based on the custom metric:

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

### Publishing the Custom Metric (Lambda)

```javascript
const { SQSClient, GetQueueAttributesCommand } = require('@aws-sdk/client-sqs');
const { ECSClient, DescribeServicesCommand } = require('@aws-sdk/client-ecs');
const { CloudWatchClient, PutMetricDataCommand } = require('@aws-sdk/client-cloudwatch');

const sqs = new SQSClient({});
const ecs = new ECSClient({});
const cw  = new CloudWatchClient({});

exports.handler = async () => {
    const queueAttrs = await sqs.send(new GetQueueAttributesCommand({
        QueueUrl: process.env.QUEUE_URL,
        AttributeNames: ['ApproximateNumberOfMessagesVisible'],
    }));
    const msgCount = parseInt(queueAttrs.Attributes.ApproximateNumberOfMessagesVisible, 10);

    const services = await ecs.send(new DescribeServicesCommand({
        cluster: process.env.CLUSTER,
        services: [process.env.SERVICE],
    }));
    const taskCount = services.services[0].runningCount;

    const backlog = taskCount === 0 ? msgCount : Math.floor(msgCount / taskCount);

    await cw.send(new PutMetricDataCommand({
        Namespace: 'Custom/WorkerScaling',
        MetricData: [{
            MetricName: 'BacklogPerInstance',
            Value: backlog,
            Unit: 'Count',
        }],
    }));
};
```

### Terraform: Scheduled Lambda for Custom Metric

```hcl
resource "aws_cloudwatch_event_rule" "backlog_metric" {
  name                = "publish-backlog-metric"
  schedule_expression = "rate(1 minute)"
}

resource "aws_cloudwatch_event_target" "backlog_lambda" {
  rule = aws_cloudwatch_event_rule.backlog_metric.name
  arn  = aws_lambda_function.backlog_publisher.arn
}

resource "aws_lambda_permission" "allow_eventbridge" {
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.backlog_publisher.function_name
  principal     = "events.amazonaws.com"
  source_arn    = aws_cloudwatch_event_rule.backlog_metric.arn
}
```

> **Cost/Performance Note:** Set `scale_in_cooldown` significantly higher than `scale_out_cooldown`. Scaling out is cheap (handle the spike). Scaling in too fast causes **flapping**: traffic spike → scale to 20 → spike subsides → scale to 2 → next batch arrives → scale to 20. A 5-minute scale-in cooldown lets transient dips pass without thrashing. Also: ECS tasks take 30–90 seconds to become healthy - factor this startup lag into your target value.
