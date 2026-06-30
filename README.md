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

