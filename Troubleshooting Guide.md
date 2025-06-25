# 🔧 Troubleshooting Guide

This guide covers common issues encountered during the AWS S3 + CloudFront portfolio setup and their solutions.

## 🚨 Common Issues and Solutions

### Issue 1: Access Denied Errors

**Problem**: Getting "Access Denied" when accessing the CloudFront URL

**Symptoms**:
- 403 Forbidden error in browser
- CloudFront returns access denied page
- Direct S3 URL also returns access denied

**Root Causes**:
1. Incorrect bucket policy configuration
2. Origin Access Control (OAC) not properly configured
3. CloudFront distribution not fully deployed

**Solutions**:

#### Check Bucket Policy
```bash
# Verify bucket policy is correctly applied
aws s3api get-bucket-policy --bucket your-bucket-name
```

#### Verify OAC Configuration
1. Go to CloudFront Console → Distributions
2. Select your distribution → Origins tab
3. Ensure Origin Access Control is selected (not Legacy OAI)
4. Verify the OAC name matches your bucket policy

#### Wait for Deployment
```bash
# Check distribution status
aws cloudfront get-distribution --id YOUR-DISTRIBUTION-ID
```
Status should be "Deployed" (not "In Progress")

---

### Issue 2: Default Root Object Not Loading

**Problem**: CloudFront URL shows directory listing or 404 error instead of your portfolio

**Symptoms**:
- Accessing `https://your-distribution.cloudfront.net/` doesn't load index.html
- Have to manually type `/index.html` to see the site

**Solution**:
1. Go to CloudFront Console → Distributions
2. Select your distribution → General → Settings → Edit
3. Set "Default Root Object" to `index.html`
4. Save changes and wait for deployment

---

### Issue 3: Cache Issues

**Problem**: Changes to your website aren't showing up

**Symptoms**:
- Updated files in S3 but old content still shows
- CSS/JS changes not reflecting
- `X-Cache: Hit from cloudfront` in response headers

**Solutions**:

#### Create Cache Invalidation
```bash
# Invalidate all files
aws cloudfront create-invalidation \
    --distribution-id YOUR-DISTRIBUTION-ID \
    --paths "/*"

# Invalidate specific files
aws cloudfront create-invalidation \
    --distribution-id YOUR-DISTRIBUTION-ID \
    --paths "/index.html" "/style.css"
```

#### Check Cache Headers
```bash
# Check if content is being cached
curl -I https://your-distribution.cloudfront.net/
# Look for X-Cache header
```

---

### Issue 4: HTTPS Not Working

**Problem**: Site loads over HTTP but not HTTPS

**Symptoms**:
- Browser shows "Not Secure" warning
- HTTP version works but HTTPS doesn't
- Certificate errors

**Solutions**:

#### Check Viewer Protocol Policy
1. CloudFront Console → Distributions → Behaviors
2. Select Default behavior → Edit
3. Set "Viewer Protocol Policy" to "Redirect HTTP to HTTPS"

#### Verify SSL Certificate
- CloudFront automatically provides SSL certificate
- Check distribution settings → General → SSL Certificate

---

### Issue 5: Slow Loading Times

**Problem**: Website loads slowly despite CloudFront

**Symptoms**:
- High Time to First Byte (TTFB)
- Slow image loading
- Poor Lighthouse performance scores

**Solutions**:

#### Optimize Cache Settings
```json
{
  "Cache-Control": "public, max-age=86400",
  "Expires": "Sat, 26 Jul 1997 05:00:00 GMT"
}
```

#### Check Origin Location
- Ensure S3 bucket is in us-east-1 for best performance
- Consider using CloudFront's global network

#### Optimize Assets
- Compress images
- Minify CSS/JS
- Enable Gzip compression in CloudFront

---

### Issue 6: Bucket Policy Errors

**Problem**: Unable to apply bucket policy

**Symptoms**:
- "Invalid policy" errors
- "Access denied" when applying policy
- Bucket policy validation failures

**Common Fixes**:

#### ARN Format Issues
```json
// ❌ Incorrect
"Resource": "arn:aws:s3:::bucket-name"

// ✅ Correct for objects
"Resource": "arn:aws:s3:::bucket-name/*"
```

#### Account ID Format
```json
// ❌ Incorrect
"AWS:SourceArn": "arn:aws:cloudfront:123456789012:distribution/E1234567890"

// ✅ Correct
"AWS:SourceArn": "arn:aws:cloudfront::123456789012:distribution/E1234567890"
```

---

### Issue 7: Origin Access Control (OAC) Problems

**Problem**: OAC not working properly

**Symptoms**:
- Access denied errors persist
- S3 bucket accessible directly (security issue)
- CloudFront can't access S3 content

**Solutions**:

#### Create New OAC
1. CloudFront Console → Origin Access Control
2. Create new OAC with these settings:
   - **Name**: `portfolio-oac`
   - **Description**: `OAC for portfolio S3 bucket`
   - **Origin type**: S3
   - **Signing behavior**: Sign requests

#### Update Distribution Origin
1. Go to your distribution → Origins
2. Edit the S3 origin
3. Select the new OAC
4. Update the origin

#### Update Bucket Policy
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
      "Resource": "arn:aws:s3:::YOUR-BUCKET-NAME/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::YOUR-ACCOUNT-ID:distribution/YOUR-DISTRIBUTION-ID"
        }
      }
    }
  ]
}
```

---

## 🔍 Debugging Commands

### Check S3 Configuration
```bash
# List bucket contents
aws s3 ls s3://your-bucket-name

# Check bucket public access settings
aws s3api get-public-access-block --bucket your-bucket-name

# View bucket policy
aws s3api get-bucket-policy --bucket your-bucket-name

# Check bucket location
aws s3api get-bucket-location --bucket your-bucket-name
```

### Check CloudFront Configuration
```bash
# Get distribution details
aws cloudfront get-distribution --id YOUR-DISTRIBUTION-ID

# List all distributions
aws cloudfront list-distributions

# Check invalidation status
aws cloudfront list-invalidations --distribution-id YOUR-DISTRIBUTION-ID

# Get distribution config
aws cloudfront get-distribution-config --id YOUR-DISTRIBUTION-ID
```

### Test Connectivity
```bash
# Test direct S3 access (should fail)
curl -I https://your-bucket.s3.amazonaws.com/index.html

# Test CloudFront access (should succeed)
curl -I https://your-distribution.cloudfront.net/

# Check specific headers
curl -H "Accept-Encoding: gzip" -I https://your-distribution.cloudfront.net/

# Test with verbose output
curl -v https://your-distribution.cloudfront.net/
```

---

## 📊 Performance Diagnostics

### Use Browser Developer Tools
1. **Network Tab**: Check load times and cache status
2. **Security Tab**: Verify HTTPS certificate
3. **Lighthouse**: Run performance audit
4. **Console Tab**: Look for JavaScript errors

### CloudWatch Metrics
- Monitor CloudFront requests and cache hit ratio
- Check S3 request metrics
- Set up alarms for unusual traffic patterns

### Performance Testing Commands
```bash
# Test page load speed
curl -w "@curl-format.txt" -o /dev/null -s https://your-distribution.cloudfront.net/

# Create curl-format.txt with:
# time_namelookup:    %{time_namelookup}\n
# time_connect:       %{time_connect}\n
# time_appconnect:    %{time_appconnect}\n
# time_pretransfer:   %{time_pretransfer}\n
# time_redirect:      %{time_redirect}\n
# time_starttransfer: %{time_starttransfer}\n
# time_total:         %{time_total}\n
```

---

## 🔐 Security Diagnostics

### Verify Security Settings
```bash
# Check if bucket is public (should return error)
aws s3api get-bucket-acl --bucket your-bucket-name

# Verify public access block
aws s3api get-public-access-block --bucket your-bucket-name

# Test security headers
curl -I https://your-distribution.cloudfront.net/
```

### Expected Security Headers
- `X-Frame-Options: DENY`
- `X-Content-Type-Options: nosniff`
- `Referrer-Policy: strict-origin-when-cross-origin`
- `Strict-Transport-Security: max-age=31536000`

---

## 📋 Troubleshooting Checklist

### Before You Start
- [ ] AWS CLI configured with correct credentials
- [ ] Correct AWS region selected
- [ ] Sufficient IAM permissions

### S3 Configuration
- [ ] Bucket exists and accessible
- [ ] Files uploaded successfully
- [ ] Public access blocked
- [ ] Bucket policy applied correctly
- [ ] Encryption enabled

### CloudFront Configuration
- [ ] Distribution created successfully
- [ ] Origin pointing to correct S3 bucket
- [ ] Origin Access Control configured
- [ ] Default root object set to `index.html`
- [ ] HTTPS redirect enabled
- [ ] Distribution status is "Deployed"

### Testing
- [ ] Direct S3 access blocked (returns 403)
- [ ] CloudFront URL accessible
- [ ] HTTPS working correctly
- [ ] Cache headers present
- [ ] Performance acceptable

---

## 🆘 Still Having Issues?

If you're still experiencing problems:

1. **Check AWS Service Health**: Visit [AWS Status Page](https://status.aws.amazon.com/)
2. **Review CloudTrail Logs**: Look for API call errors
3. **Contact AWS Support**: For persistent issues
4. **Community Forums**: AWS re:Post for community help

### Useful Resources
- [AWS CloudFront Documentation](https://docs.aws.amazon.com/cloudfront/)
- [S3 Static Website Hosting Guide](https://docs.aws.amazon.com/s3/latest/userguide/WebsiteHosting.html)
- [CloudFront Troubleshooting](https://docs.aws.amazon.com/cloudfront/latest/developerguide/troubleshooting.html)
- [AWS CLI Reference](https://docs.aws.amazon.com/cli/)

---

## 📝 Prevention Tips

1. **Always test changes** in a staging environment first
2. **Document your configurations** for easy reference
3. **Monitor costs** with AWS Budgets
4. **Regular backups** of your S3 content
5. **Keep policies minimal** - grant only necessary permissions
6. **Use version control** for your website files
7. **Set up monitoring** with CloudWatch alarms

## 🔄 Recovery Procedures

### If You Break Something
1. **Don't panic** - most issues are reversible
2. **Check recent changes** - roll back if needed
3. **Use AWS CLI** to verify configurations
4. **Recreate resources** if necessary (they're cheap/free)
5. **Document the fix** for future reference

Remember: Most issues are configuration-related and can be resolved by carefully reviewing the settings and comparing with working examples!