# learn-aws-athena
### Step 1: Enable ALB access logging and point it to an S3 bucket.
```
In the AWS Console (or via CLI/Terraform), go to your Application Load Balancer's attributes:

1. Open EC2 Console → Load Balancers → select your ALB → Attributes tab (or "Edit attributes")
2. Find "Monitoring" section → enable "Access logs"
3. Specify the S3 bucket name and an optional prefix (e.g., `alb-logs/`)
4. The bucket must be in the **same region** as the ALB, and needs a bucket policy granting the ELB service account in that region permission to write to it (AWS gives you a one-click option to auto-create this policy, or you can attach it manually)

Via CLI, this looks like:
```
```bash
aws elbv2 modify-load-balancer-attributes \
  --load-balancer-arn arn:aws:elasticloadbalancing:REGION:ACCOUNT_ID:loadbalancer/app/NAME/ID \
  --attributes Key=access_logs.s3.enabled,Value=true \
              Key=access_logs.s3.bucket,Value=your-log-bucket \
              Key=access_logs.s3.prefix,Value=alb-logs
```

Once enabled, logs will start landing in S3 as gzip files under a path like:

```
s3://your-log-bucket/alb-logs/AWSLogs/ACCOUNT_ID/elasticloadbalancing/REGION/yyyy/mm/dd/
```
Let me know when you're ready and I'll walk you through step 2 (creating the Athena table over this data).
### Step 2: Create an Athena table over the log data.

Open Athena in the console (make sure the workgroup's query result location is set to an S3 bucket), select the right database (or create one), and run this DDL — it's AWS's standard ALB log schema using a regex SerDe:

```sql
CREATE EXTERNAL TABLE IF NOT EXISTS alb_logs (
    type string,
    time string,
    elb string,
    client_ip string,
    client_port int,
    target_ip string,
    target_port int,
    request_processing_time double,
    target_processing_time double,
    response_processing_time double,
    elb_status_code int,
    target_status_code string,
    received_bytes bigint,
    sent_bytes bigint,
    request_verb string,
    request_url string,
    request_proto string,
    user_agent string,
    ssl_cipher string,
    ssl_protocol string,
    target_group_arn string,
    trace_id string,
    domain_name string,
    chosen_cert_arn string,
    matched_rule_priority string,
    request_creation_time string,
    actions_executed string,
    redirect_url string,
    lambda_error_reason string,
    target_port_list string,
    target_status_code_list string,
    classification string,
    classification_reason string
)
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.RegexSerDe'
WITH SERDEPROPERTIES (
    'serialization.format' = '1',
    'input.regex' = '([^ ]*) ([^ ]*) ([^ ]*) ([^ ]*):([0-9]*) ([^ ]*)[:-]([0-9]*) ([-.0-9]*) ([-.0-9]*) ([-.0-9]*) (|[-0-9]*) (-|[-0-9]*) ([-0-9]*) ([-0-9]*) \"([^ ]*) ([^ ]*) (- |[^ ]*)\" \"([^\"]*)\" ([A-Z0-9-_]+) ([A-Za-z0-9.-]*) ([^ ]*) \"([^\"]*)\" \"([^\"]*)\" \"([^\"]*)\" ([-.0-9]*) ([^ ]*) \"([^\"]*)\" \"([^\"]*)\" \"([^ ]*)\" \"([^ ]*)\" \"([^ ]*)\" \"([^ ]*)\" \"([^ ]*)\"'
)
LOCATION 's3://your-log-bucket/alb-logs/AWSLogs/ACCOUNT_ID/elasticloadbalancing/REGION/';
```

Replace the `LOCATION` with your actual bucket/prefix/account/region path. Note this points at the root above the date partitions, so without partitioning Athena will scan every file recursively (fine for testing, costly at scale).

Once that runs successfully, you can sanity-check it with:

```sql
SELECT * FROM alb_logs LIMIT 10;
```
Let me know when that works and I'll give you step 3: adding partitioning (by date) so your queries are fast and cheap instead of scanning the whole bucket every time.
