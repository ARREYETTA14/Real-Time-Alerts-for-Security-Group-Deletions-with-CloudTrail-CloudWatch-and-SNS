# Real-Time-Alerts-for-Security-Group-Deletions-with-CloudTrail-CloudWatch-and-SNS
When a user deletes a Security Group, CloudTrail records it, CloudWatch picks the event up from the CloudTrail log stream, a metric filter increments, an alarm fires, and SNS emails you.


## 1) Create an SNS topic & confirm email subscription

- Console: **SNS** → **Topics** → **Create topic** → **Standard**

	- Name: ``cloudtrail-delete-alerts``

- Click the topic → Create subscription → Protocol: Email → Endpoint: you@example.com

- Open your email and confirm the subscription link. (If not confirmed, no emails.)

## 2) Create (or pick) a CloudTrail trail that streams to CloudWatch Logs

- You must send CloudTrail events to CloudWatch Logs to get near-real-time metric filters and alarms.

- **Console** → **CloudTrail** → **Trails** → **Create trail**

- Name: **DemoTrail**

- **Storage location**: create / pick an S3 bucket (CloudTrail requires S3).

- **Apply trail to all regions** (recommended) — makes sure you don’t miss events.

- In **CloudWatch Logs**, click **Configure**, choose/create a **Log group** name, e.g. ``/aws/cloudtrail/DemoTrail``.

- For the role: if the console can create the role for you choose ``Create new role``. 

- Create Trail and **Start logging**.

## 3) Create a CloudWatch Logs Metric Filter that looks for the DeleteSecurityGroup event
- **CloudWatch** → **Logs** → **Log groups** → open ``/aws/cloudtrail/DemoTrail``.

- Click **Create metric filter**.

- Enter the filter pattern:

```text
{ ($.eventName = "DeleteSecurityGroup") }
```

Explanation: CloudTrail log entries are JSON; eventName contains API like DeleteSecurityGroup.

## 4) Click **Next**. Configure metric:

- Metric namespace: ``CloudTrailSecurity``

- Metric name: ``DeleteSecurityGroupCount``

- Metric value: ``1``

## 5) Create the metric filter.

(If you want to catch multiple delete actions, you can use an OR pattern:

```text
{ ($.eventName = "DeleteSecurityGroup") || ($.eventName = "DeleteBucket") || ($.eventName = "TerminateInstances") }
```)
```

## 6) Create a CloudWatch Alarm on that metric
- **CloudWatch → Alarms → Create alarm**.  
- **Select metric** → choose `Logs` → find your  → `DeleteSecurityGroupCount`.  
- Define condition:
   - Statistic: `Sum`
   - Period: `1 minute` (or 5 minutes)
   - Threshold: `>= 1` (i.e., whenever at least one delete occurs in the period)
   - Datapoints to alarm: 1 out of 1 (so it triggers on the first match)
- Actions:
   - Add notification → choose SNS topic `cloudtrail-delete-alerts`.
- Name the alarm e.g. `Alert-DeleteSecurityGroup`.

**Note:** Alarms fire fast once CloudWatch metric receives data (CloudTrail → CloudWatch logs ingestion latency is typically seconds to a couple minutes).

---

## 7) Test the pipeline (safe test)
We will create a temporary SG and then delete it to trigger CloudTrail.

