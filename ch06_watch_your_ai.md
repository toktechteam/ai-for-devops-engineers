# Chapter 6: Watch Your AI (Token Dashboard and a Budget Alarm)

💸 **Cost:** a few cents at most (CloudWatch custom dashboards are free up to 3 per month; alarms are about $0.10/month each while they exist; SNS email delivery is free)
⏱ **Time:** 25–30 minutes
🧹 **Cleanup:** delete the alarm, the dashboard and the SNS topic at the end of this chapter

**The purpose of this chapter, in one sentence:** stop relying on a number printed in your terminal, and build the dashboard-plus-alarm pattern that tells you — by email, automatically — if an AI feature's cost starts climbing.

**Why anyone should spend 25–30 minutes on this:** every chapter before this one printed a token count after each call, but that number disappears the moment you close the terminal. In a real team, nobody watches a terminal for that; they watch a dashboard and get paged when a number crosses a line. This is also one of the first things a DevOps/SRE interview checks for: can you monitor a service you didn't build the billing logic for?

**By the end of this chapter, you will have:**
- Found the CloudWatch metrics that Amazon Bedrock publishes automatically for every model call — no code changes needed
- Built a CloudWatch dashboard that shows token usage, invocation count, latency and errors for your Nova Micro calls
- Created a CloudWatch alarm that emails you when daily token usage crosses a budget you set
- Proven the alarm actually fires, by temporarily lowering the threshold and generating real traffic

🎥 [VIDEO ch06 | QR code + short link | ~7 min | unlisted]

## Why this matters

Every chapter so far has called Bedrock and printed the token usage from the response — that number only exists in your terminal, for one call, at the moment you looked. In a real team, nobody checks a terminal to see if an AI feature's cost is climbing. They check a dashboard, and they get paged when a number crosses a line.

This is also one of the first things a DevOps/SRE role expects you to know: how do you watch a service you didn't build the billing logic for? AWS already meters every Bedrock call and publishes the numbers to CloudWatch — you don't add logging code, you just know where to look and how to alert on it. This chapter is that skill, applied to the labs you already built.

## Concept

**DevOps analogy:** this is the same pattern as watching EC2 CPU or an ALB's request count — a managed AWS service emits metrics to CloudWatch on its own, you build a dashboard to see trends, and you set an alarm so you find out from email instead of from a surprise bill.

**What Bedrock publishes, automatically, per model:**

| Metric | What it tells you |
|---|---|
| `Invocations` | How many times a model was called |
| `InputTokenCount` / `OutputTokenCount` | The token counts that drive cost — this is what you're watching in this chapter |
| `InvocationLatency` | How long calls take (useful for spotting a slow model or a slow region) |
| `InvocationClientErrors` | Calls that failed because of how you called it (bad input, throttling from your side) |
| `InvocationServerErrors` | Calls that failed on AWS's side |
| `InvocationThrottles` | Calls rejected because you hit a rate limit |

These live in the `AWS/Bedrock` namespace, broken down by a `ModelId` dimension — so you can watch Nova Micro specifically, separate from any other model you might add later.

**Why a token alarm instead of a dollar alarm?** AWS Budgets and Cost Explorer show your bill, but with a delay of several hours, and they don't break "Bedrock" down by model without extra setup. Token count is the number that turns directly into cost, it updates within minutes of a real call, and it's the number your own code already prints — so it's the fastest signal that something is calling the model far more than expected (a retry loop with no backoff, a script left running, or a bug that calls the model once per log line instead of once per batch).

**Where this fits with the free tier:** your first $100–$200 of AWS credit (Chapter 0) absorbs the cost of running this lab many times over. The point of this chapter is the pattern — dashboard plus alarm — which is exactly what you'd size properly with real budget numbers once this becomes a real feature with real traffic.

## Hands-on

All steps run in **CloudShell** (region `us-east-1`), signed in as the `opskart-admin` user from Chapter 0. `opskart-admin` has `AdministratorAccess`, so no new IAM role is needed for this chapter. (If you were doing this from a scoped role instead, it would need `cloudwatch:PutDashboard`, `cloudwatch:PutMetricAlarm`, `cloudwatch:GetMetricData`, `cloudwatch:ListMetrics`, and `sns:CreateTopic`/`sns:Subscribe`.)

### Step 1: Generate some traffic to watch

Bedrock only publishes a metric data point when a call actually happens. Reuse `ask.py` from Chapter 2 (on the `opskart-bedrock-lab` EC2 instance, if it's still running) or `safe_explain.py` from Chapter 5 — anything that calls `converse()` a handful of times works. If neither is still around, this one-liner from CloudShell does it (CloudShell's `opskart-admin` credentials can call Bedrock directly):

```bash
for i in 1 2 3 4 5; do
  aws bedrock-runtime converse \
    --region us-east-1 \
    --model-id us.amazon.nova-micro-v1:0 \
    --messages '[{"role":"user","content":[{"text":"In one short sentence, what does a Kubernetes readiness probe do?"}]}]' \
    --query 'output.message.content[0].text' --output text
  sleep 2
done
```

**Expected output:** five short one-sentence answers, printed one after another.

Bedrock metrics can take a few minutes to show up in CloudWatch after a call. Keep that in mind for every step below — if a metric or graph looks empty, wait 2–3 minutes and try again.

### Step 2: Confirm the metrics exist

```bash
aws cloudwatch list-metrics \
  --region us-east-1 \
  --namespace AWS/Bedrock \
  --query 'Metrics[].MetricName' --output text | tr '\t' '\n' | sort -u
```

**Expected output** (order may differ; you may see fewer if you haven't hit every code path yet, e.g. no errors):

```
InputTokenCount
InvocationClientErrors
InvocationLatency
InvocationServerErrors
InvocationThrottles
Invocations
OutputTokenCount
```

If this comes back empty, you ran Step 1 in a different region, or you need to wait a few more minutes.

Look at the raw token numbers for the last hour before building the dashboard:

```bash
aws cloudwatch get-metric-statistics \
  --region us-east-1 \
  --namespace AWS/Bedrock \
  --metric-name InputTokenCount \
  --dimensions Name=ModelId,Value=us.amazon.nova-micro-v1:0 \
  --start-time "$(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S)" \
  --end-time "$(date -u +%Y-%m-%dT%H:%M:%S)" \
  --period 300 \
  --statistics Sum \
  --query 'Datapoints[].[Timestamp,Sum]' --output table
```

**Expected output:** a small table with one row per 5-minute window that had traffic, and a `Sum` column showing input token counts (small numbers — Nova Micro prompts in this book are short).

### Step 3: Create an SNS topic for alerts

This is where the budget alarm will send email.

```bash
mkdir -p ~/ch06 && cd ~/ch06

TOPIC_ARN=$(aws sns create-topic \
  --region us-east-1 \
  --name opskart-ai-budget-alerts \
  --query 'TopicArn' --output text)

echo "export TOPIC_ARN=$TOPIC_ARN" > ~/ch06/sns.env
echo "Topic ARN: $TOPIC_ARN"
```

**Expected output:**

```
Topic ARN: arn:aws:sns:us-east-1:123456789012:opskart-ai-budget-alerts
```

Subscribe your own email address (replace with yours):

```bash
aws sns subscribe \
  --region us-east-1 \
  --topic-arn "$TOPIC_ARN" \
  --protocol email \
  --notification-endpoint you@example.com
```

**Expected output:**

```json
{
    "SubscriptionArn": "pending confirmation"
}
```

AWS immediately emails that address a confirmation link. **Open the email and click "Confirm subscription"** — the alarm cannot notify you until you do this.

```bash
aws sns list-subscriptions-by-topic \
  --region us-east-1 \
  --topic-arn "$TOPIC_ARN" \
  --query 'Subscriptions[].SubscriptionArn' --output text
```

**Expected output:** after confirming, this shows a real ARN (a long string), not `PendingConfirmation`.

### Step 4: Build the CloudWatch dashboard

```bash
cat > ~/ch06/dashboard.json <<'EOF'
{
  "widgets": [
    {
      "type": "metric",
      "x": 0, "y": 0, "width": 12, "height": 6,
      "properties": {
        "title": "Nova Micro tokens per hour",
        "region": "us-east-1",
        "view": "timeSeries",
        "stacked": true,
        "period": 3600,
        "metrics": [
          [ "AWS/Bedrock", "InputTokenCount", "ModelId", "us.amazon.nova-micro-v1:0", { "stat": "Sum", "label": "Input tokens" } ],
          [ "AWS/Bedrock", "OutputTokenCount", "ModelId", "us.amazon.nova-micro-v1:0", { "stat": "Sum", "label": "Output tokens" } ]
        ]
      }
    },
    {
      "type": "metric",
      "x": 12, "y": 0, "width": 12, "height": 6,
      "properties": {
        "title": "Invocations per hour",
        "region": "us-east-1",
        "view": "timeSeries",
        "stacked": false,
        "period": 3600,
        "metrics": [
          [ "AWS/Bedrock", "Invocations", "ModelId", "us.amazon.nova-micro-v1:0", { "stat": "Sum", "label": "Calls" } ]
        ]
      }
    },
    {
      "type": "metric",
      "x": 0, "y": 6, "width": 12, "height": 6,
      "properties": {
        "title": "Latency (ms)",
        "region": "us-east-1",
        "view": "timeSeries",
        "stacked": false,
        "period": 3600,
        "metrics": [
          [ "AWS/Bedrock", "InvocationLatency", "ModelId", "us.amazon.nova-micro-v1:0", { "stat": "Average", "label": "Average" } ],
          [ "AWS/Bedrock", "InvocationLatency", "ModelId", "us.amazon.nova-micro-v1:0", { "stat": "p99", "label": "p99" } ]
        ]
      }
    },
    {
      "type": "metric",
      "x": 12, "y": 6, "width": 12, "height": 6,
      "properties": {
        "title": "Errors and throttles",
        "region": "us-east-1",
        "view": "timeSeries",
        "stacked": false,
        "period": 3600,
        "metrics": [
          [ "AWS/Bedrock", "InvocationClientErrors", "ModelId", "us.amazon.nova-micro-v1:0", { "stat": "Sum", "label": "Client errors" } ],
          [ "AWS/Bedrock", "InvocationServerErrors", "ModelId", "us.amazon.nova-micro-v1:0", { "stat": "Sum", "label": "Server errors" } ],
          [ "AWS/Bedrock", "InvocationThrottles", "ModelId", "us.amazon.nova-micro-v1:0", { "stat": "Sum", "label": "Throttles" } ]
        ]
      }
    }
  ]
}
EOF

aws cloudwatch put-dashboard \
  --region us-east-1 \
  --dashboard-name opskart-ai-lab \
  --dashboard-body file://~/ch06/dashboard.json
```

**Expected output:**

```json
{
    "DashboardValidationMessages": []
}
```

An empty `DashboardValidationMessages` list means it was accepted with no warnings.

Open it in the console: **CloudWatch → Dashboards → opskart-ai-lab**. If a panel is empty, widen the time range (top right) to 3 hours, since Step 1's traffic may already have scrolled out of the default 1-hour view.

### Step 5: Create the daily token budget alarm

This alarm adds input and output tokens together with a metric math expression, sums them over a full day, and fires if the total goes over a threshold you choose.

```bash
source ~/ch06/sns.env

cat > ~/ch06/alarm-metrics.json <<'EOF'
[
  {
    "Id": "input_tokens",
    "MetricStat": {
      "Metric": {
        "Namespace": "AWS/Bedrock",
        "MetricName": "InputTokenCount",
        "Dimensions": [ { "Name": "ModelId", "Value": "us.amazon.nova-micro-v1:0" } ]
      },
      "Period": 86400,
      "Stat": "Sum"
    },
    "ReturnData": false
  },
  {
    "Id": "output_tokens",
    "MetricStat": {
      "Metric": {
        "Namespace": "AWS/Bedrock",
        "MetricName": "OutputTokenCount",
        "Dimensions": [ { "Name": "ModelId", "Value": "us.amazon.nova-micro-v1:0" } ]
      },
      "Period": 86400,
      "Stat": "Sum"
    },
    "ReturnData": false
  },
  {
    "Id": "total_tokens",
    "Expression": "input_tokens + output_tokens",
    "Label": "Total tokens per day",
    "ReturnData": true
  }
]
EOF

aws cloudwatch put-metric-alarm \
  --region us-east-1 \
  --alarm-name opskart-ai-daily-token-budget \
  --alarm-description "Fires when Nova Micro token use crosses the daily lab budget" \
  --metrics file://~/ch06/alarm-metrics.json \
  --comparison-operator GreaterThanThreshold \
  --threshold 20000 \
  --evaluation-periods 1 \
  --treat-missing-data notBreaching \
  --alarm-actions "$TOPIC_ARN" \
  --ok-actions "$TOPIC_ARN"
```

**Expected output:** the command prints nothing on success.

```bash
aws cloudwatch describe-alarms \
  --region us-east-1 \
  --alarm-names opskart-ai-daily-token-budget \
  --query 'MetricAlarms[0].{Name:AlarmName,State:StateValue,Threshold:Threshold}' --output table
```

**Expected output:**

```
---------------------------------------------------------
|                    DescribeAlarms                     |
+--------------------------------------+-------+---------+
|  Name                                | State | Threshold|
+--------------------------------------+-------+---------+
|  opskart-ai-daily-token-budget       | OK    | 20000.0 |
+--------------------------------------+-------+---------+
```

`20000` tokens/day is a placeholder for a lab that runs a handful of calls a day. Check the [Amazon Bedrock pricing page](https://aws.amazon.com/bedrock/pricing/) for Nova Micro's current per-1,000-token rates, decide what a day of real traffic should cost, and set the threshold from that number, not from this book.

### Step 6: Prove the alarm actually fires

An alarm you've never seen trigger is a guess, not a safety net. Lower the threshold far below anything you'd hit by accident, generate a little traffic, and watch it flip to `ALARM`.

```bash
aws cloudwatch put-metric-alarm \
  --region us-east-1 \
  --alarm-name opskart-ai-daily-token-budget \
  --alarm-description "Fires when Nova Micro token use crosses the daily lab budget" \
  --metrics file://~/ch06/alarm-metrics.json \
  --comparison-operator GreaterThanThreshold \
  --threshold 50 \
  --evaluation-periods 1 \
  --treat-missing-data notBreaching \
  --alarm-actions "$TOPIC_ARN" \
  --ok-actions "$TOPIC_ARN"
```

Generate a few calls (reuse Step 1's loop), then wait 3–5 minutes for the metric math to evaluate on its next period, and check the state:

```bash
aws cloudwatch describe-alarms \
  --region us-east-1 \
  --alarm-names opskart-ai-daily-token-budget \
  --query 'MetricAlarms[0].StateValue' --output text
```

**Expected output (after enough traffic and enough wait):**

```
ALARM
```

You should also get an email from AWS Notifications within a minute or two of the state changing. Once you've confirmed it works, put the threshold back to a real value:

```bash
aws cloudwatch put-metric-alarm \
  --region us-east-1 \
  --alarm-name opskart-ai-daily-token-budget \
  --alarm-description "Fires when Nova Micro token use crosses the daily lab budget" \
  --metrics file://~/ch06/alarm-metrics.json \
  --comparison-operator GreaterThanThreshold \
  --threshold 20000 \
  --evaluation-periods 1 \
  --treat-missing-data notBreaching \
  --alarm-actions "$TOPIC_ARN" \
  --ok-actions "$TOPIC_ARN"
```

> **GitHub repo:** commit `dashboard.json` and `alarm-metrics.json` to `chapter-06/`, created exactly as above.

### Step 7: Cleanup

```bash
aws cloudwatch delete-alarms --region us-east-1 --alarm-names opskart-ai-daily-token-budget
aws cloudwatch delete-dashboards --region us-east-1 --dashboard-names opskart-ai-lab
aws sns delete-topic --region us-east-1 --topic-arn "$TOPIC_ARN"
```

**Expected output:** all three commands print nothing on success.

```bash
aws cloudwatch describe-alarms --region us-east-1 --alarm-names opskart-ai-daily-token-budget --query 'MetricAlarms'
```

**Expected output:** `[]` — an empty list confirms the alarm is gone.

## Troubleshooting

| Problem | Fix |
|---|---|
| `list-metrics` for `AWS/Bedrock` returns nothing | No calls have been made in this region yet, or fewer than 2–3 minutes have passed since Step 1. Re-run Step 1 and wait |
| Dashboard widgets are blank in the console | The console's default time range (often 1h or 3h) doesn't cover when you ran Step 1. Widen the time range in the top-right of the dashboard |
| `put-dashboard` fails with `InvalidParameterInput` | Usually a JSON syntax error in `dashboard.json` — re-create it with the exact heredoc in Step 4, don't hand-edit it |
| SNS subscription stuck as `PendingConfirmation` forever | You haven't clicked the confirmation link in the email yet, or it landed in spam. Re-run `sns subscribe` to get a fresh email |
| Alarm never moves to `ALARM` even with the threshold at 50 | `treat-missing-data notBreaching` means the alarm needs at least one real data point in the period; make sure Step 1's loop actually succeeded and wait a full 5-minute period, since the metric's own period here is a day (86400s) but CloudWatch still evaluates on its own schedule |
| No email arrives even though the alarm state changed | Check spam, and re-run `sns list-subscriptions-by-topic` to confirm the subscription is confirmed, not still pending |
| `AccessDenied` on any `cloudwatch:*` or `sns:*` call | You're not signed in as `opskart-admin`. Run `aws sts get-caller-identity` to check |
| Alarm threshold feels arbitrary | It is, in this lab — the whole point is you now have the pattern. Re-derive it from the actual Bedrock pricing page and your own expected daily call volume before using this on anything real |

## Interview questions

1. Where do managed AWS services like Bedrock publish their usage metrics, and how would you find out what's available for a service you've never monitored before?
2. Why watch token count instead of waiting for the AWS bill to show a cost spike?
3. What's the difference between a static-threshold alarm (like this chapter's) and an anomaly-detection alarm, and when would you reach for each?
4. You're on call and get paged by this alarm at 2 a.m. What's the first thing you'd check to tell a real incident (a bug looping on the model) from a false alarm (a one-time batch job)?

## Checklist

- [ ] Generated traffic and confirmed `AWS/Bedrock` metrics exist for Nova Micro
- [ ] Created the SNS topic and confirmed the email subscription
- [ ] Built the `opskart-ai-lab` dashboard with tokens, invocations, latency and errors
- [ ] Created the daily token budget alarm with a metric math expression
- [ ] Proved the alarm fires by lowering the threshold and watching it move to `ALARM`, then reset a real threshold
- [ ] Deleted the alarm, the dashboard and the SNS topic
