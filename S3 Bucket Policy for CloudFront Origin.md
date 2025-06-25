# S3 Bucket Policy for CloudFront Origin Access Control (OAC)

## 📋 Complete Bucket Policy

This is the exact JSON policy that should be applied to your S3 bucket to allow CloudFront access while blocking all other public access.

### Policy JSON
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCloudFrontServicePrincipal",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudfront.amazonaws.com"
      },
      "Action": "s3:GetObject",
      "Resource