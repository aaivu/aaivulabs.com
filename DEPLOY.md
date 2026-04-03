# Deploying Aaivu Labs Website to AWS S3 + Route 53

## Architecture Overview

```
User → aaivulabs.com → Route 53 (DNS) → CloudFront (CDN + HTTPS) → S3 (Static Files)
```

---

## Prerequisites

- AWS Account with admin access
- AWS CLI installed and configured (`aws configure`)
- Domain `aaivulabs.com` registered (or transferred) in Route 53
- Website files ready in the `website/` folder

### Install AWS CLI (if not installed)

```bash
# macOS
brew install awscli

# Verify
aws --version

# Configure with your credentials
aws configure
# Enter: AWS Access Key ID, Secret Access Key, Region (ap-southeast-1 or us-east-1), Output format (json)
```

---

## Step 1: Create the S3 Bucket

The bucket name **must match your domain name**.

```bash
# Create the main bucket
aws s3 mb s3://aaivulabs.com --region us-east-1

# Create www redirect bucket
aws s3 mb s3://www.aaivulabs.com --region us-east-1
```

## Step 2: Enable Static Website Hosting on S3

```bash
# Enable static hosting on the main bucket
aws s3 website s3://aaivulabs.com \
  --index-document index.html \
  --error-document error.html
```

## Step 3: Set Bucket Policy for Public Access

First, disable "Block Public Access" settings:

```bash
aws s3api put-public-access-block \
  --bucket aaivulabs.com \
  --public-access-block-configuration \
  "BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false"
```

Then apply the bucket policy:

```bash
aws s3api put-bucket-policy --bucket aaivulabs.com --policy '{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::aaivulabs.com/*"
    }
  ]
}'
```

## Step 4: Configure www Redirect Bucket

```bash
aws s3 website s3://www.aaivulabs.com \
  --redirect-all-requests-to aaivulabs.com \
  --protocol https
```

## Step 5: Upload Website Files

```bash
# From the project root directory, upload all files
aws s3 sync website/ s3://aaivulabs.com/ \
  --delete \
  --cache-control "max-age=86400"

# Set correct content types for specific files
aws s3 cp s3://aaivulabs.com/index.html s3://aaivulabs.com/index.html \
  --content-type "text/html" \
  --cache-control "max-age=3600" \
  --metadata-directive REPLACE

aws s3 cp s3://aaivulabs.com/error.html s3://aaivulabs.com/error.html \
  --content-type "text/html" \
  --cache-control "max-age=3600" \
  --metadata-directive REPLACE
```

At this point, your site should be accessible at:
`http://aaivulabs.com.s3-website-us-east-1.amazonaws.com`

---

## Step 6: Create SSL Certificate (ACM) — REQUIRED for HTTPS

> **IMPORTANT:** The certificate MUST be created in **us-east-1** (N. Virginia) for CloudFront to use it.

```bash
aws acm request-certificate \
  --domain-name aaivulabs.com \
  --subject-alternative-names "*.aaivulabs.com" \
  --validation-method DNS \
  --region us-east-1
```

This returns a `CertificateArn`. Save it — you'll need it later.

```bash
# Example output:
# "CertificateArn": "arn:aws:acm:us-east-1:123456789:certificate/abc-123-def-456"
```

### Validate the Certificate via DNS

```bash
# Get the DNS validation records
aws acm describe-certificate \
  --certificate-arn <YOUR_CERTIFICATE_ARN> \
  --region us-east-1 \
  --query 'Certificate.DomainValidationOptions'
```

This gives you a CNAME record to add. Go to **Route 53 → Hosted Zone → aaivulabs.com** and add the CNAME record shown. Or use the CLI:

```bash
# Get your hosted zone ID
aws route53 list-hosted-zones-by-name --dns-name aaivulabs.com \
  --query 'HostedZones[0].Id' --output text
```

```bash
# Add the validation CNAME (replace values from describe-certificate output)
aws route53 change-resource-record-sets \
  --hosted-zone-id <HOSTED_ZONE_ID> \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "<CNAME_NAME_FROM_ACM>",
        "Type": "CNAME",
        "TTL": 300,
        "ResourceRecords": [{"Value": "<CNAME_VALUE_FROM_ACM>"}]
      }
    }]
  }'
```

Wait for validation (usually 5-30 minutes):

```bash
aws acm wait certificate-validated \
  --certificate-arn <YOUR_CERTIFICATE_ARN> \
  --region us-east-1
```

---

## Step 7: Create CloudFront Distribution

Create a file called `cloudfront-config.json`:

```json
{
  "CallerReference": "aaivulabs-2026",
  "Aliases": {
    "Quantity": 2,
    "Items": ["aaivulabs.com", "www.aaivulabs.com"]
  },
  "DefaultRootObject": "index.html",
  "Origins": {
    "Quantity": 1,
    "Items": [
      {
        "Id": "S3-aaivulabs.com",
        "DomainName": "aaivulabs.com.s3-website-us-east-1.amazonaws.com",
        "CustomOriginConfig": {
          "HTTPPort": 80,
          "HTTPSPort": 443,
          "OriginProtocolPolicy": "http-only"
        }
      }
    ]
  },
  "DefaultCacheBehavior": {
    "TargetOriginId": "S3-aaivulabs.com",
    "ViewerProtocolPolicy": "redirect-to-https",
    "AllowedMethods": {
      "Quantity": 2,
      "Items": ["GET", "HEAD"]
    },
    "CachePolicyId": "658327ea-f89d-4fab-a63d-7e88639e58f6",
    "Compress": true
  },
  "CustomErrorResponses": {
    "Quantity": 1,
    "Items": [
      {
        "ErrorCode": 404,
        "ResponseCode": "404",
        "ResponsePagePath": "/error.html",
        "ErrorCachingMinTTL": 300
      }
    ]
  },
  "ViewerCertificate": {
    "ACMCertificateArn": "<YOUR_CERTIFICATE_ARN>",
    "SSLSupportMethod": "sni-only",
    "MinimumProtocolVersion": "TLSv1.2_2021"
  },
  "Enabled": true,
  "Comment": "Aaivu Labs Website",
  "PriceClass": "PriceClass_All",
  "HttpVersion": "http2and3"
}
```

> Replace `<YOUR_CERTIFICATE_ARN>` with the ARN from Step 6.

```bash
aws cloudfront create-distribution \
  --distribution-config file://cloudfront-config.json
```

Save the **Distribution ID** and **Domain Name** (e.g., `d1234abcdef.cloudfront.net`) from the output.

Wait for deployment (takes 10-20 minutes):

```bash
aws cloudfront wait distribution-deployed \
  --id <DISTRIBUTION_ID>
```

---

## Step 8: Configure Route 53 DNS Records

Point your domain to the CloudFront distribution.

```bash
# Get your hosted zone ID
ZONE_ID=$(aws route53 list-hosted-zones-by-name \
  --dns-name aaivulabs.com \
  --query 'HostedZones[0].Id' \
  --output text)

echo "Zone ID: $ZONE_ID"
```

### Create A record for root domain (aaivulabs.com)

```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id $ZONE_ID \
  --change-batch '{
    "Changes": [
      {
        "Action": "UPSERT",
        "ResourceRecordSet": {
          "Name": "aaivulabs.com",
          "Type": "A",
          "AliasTarget": {
            "HostedZoneId": "Z2FDTNDATAQYW2",
            "DNSName": "<CLOUDFRONT_DOMAIN>.cloudfront.net",
            "EvaluateTargetHealth": false
          }
        }
      }
    ]
  }'
```

> **Note:** `Z2FDTNDATAQYW2` is the fixed hosted zone ID for ALL CloudFront distributions. Do not change this.

### Create A record for www subdomain

```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id $ZONE_ID \
  --change-batch '{
    "Changes": [
      {
        "Action": "UPSERT",
        "ResourceRecordSet": {
          "Name": "www.aaivulabs.com",
          "Type": "A",
          "AliasTarget": {
            "HostedZoneId": "Z2FDTNDATAQYW2",
            "DNSName": "<CLOUDFRONT_DOMAIN>.cloudfront.net",
            "EvaluateTargetHealth": false
          }
        }
      }
    ]
  }'
```

> Replace `<CLOUDFRONT_DOMAIN>` with your actual CloudFront domain (e.g., `d1234abcdef`).

---

## Step 9: Verify Everything

```bash
# Check DNS propagation
dig aaivulabs.com
dig www.aaivulabs.com

# Test the website
curl -I https://aaivulabs.com
curl -I https://www.aaivulabs.com

# Should show 200 OK and redirect www → root
```

Visit `https://aaivulabs.com` — your site should be live!

---

## Updating the Website

When you make changes to the website files:

```bash
# 1. Upload changed files
aws s3 sync website/ s3://aaivulabs.com/ --delete

# 2. Invalidate CloudFront cache to see changes immediately
aws cloudfront create-invalidation \
  --distribution-id <DISTRIBUTION_ID> \
  --paths "/*"
```

---

## Quick Deployment Script

Save this as `deploy.sh` in the project root:

```bash
#!/bin/bash
set -e

BUCKET="aaivulabs.com"
DISTRIBUTION_ID="<YOUR_DISTRIBUTION_ID>"  # Replace after setup

echo "Uploading files to S3..."
aws s3 sync website/ s3://$BUCKET/ \
  --delete \
  --cache-control "max-age=86400"

# Fix content types for HTML files
aws s3 cp s3://$BUCKET/index.html s3://$BUCKET/index.html \
  --content-type "text/html" \
  --cache-control "max-age=3600" \
  --metadata-directive REPLACE

aws s3 cp s3://$BUCKET/error.html s3://$BUCKET/error.html \
  --content-type "text/html" \
  --cache-control "max-age=3600" \
  --metadata-directive REPLACE

echo "Invalidating CloudFront cache..."
aws cloudfront create-invalidation \
  --distribution-id $DISTRIBUTION_ID \
  --paths "/*"

echo "Deployment complete! Changes will be live within a few minutes."
```

```bash
chmod +x deploy.sh
./deploy.sh
```

---

## Cost Estimate (Monthly)

| Service     | Cost         |
|-------------|-------------|
| S3          | ~$0.50      |
| CloudFront  | ~$1-5       |
| Route 53    | ~$0.50      |
| ACM (SSL)   | Free        |
| **Total**   | **~$2-6/mo** |

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| "Access Denied" on S3 | Check bucket policy and public access settings (Step 3) |
| Certificate stuck on "Pending" | Verify the CNAME record was added correctly in Route 53 |
| CloudFront returning old content | Run cache invalidation (`aws cloudfront create-invalidation`) |
| www not redirecting | Ensure www bucket has redirect config and CloudFront has both aliases |
| "The request could not be satisfied" | CloudFront is still deploying — wait 15-20 minutes |
| Mixed content warnings | All assets use CDN URLs (https) — already handled in our template |

---

## File Structure

```
aaivulabs.com/
├── website/
│   ├── index.html            ← Main website
│   ├── error.html            ← 404 error page
│   └── assets/
│       ├── logo.png          ← Aaivu Labs logo (PNG)
│       ├── logo.svg          ← Aaivu Labs logo (SVG)
│       └── logo-labs.svg     ← Labs variant (SVG)
├── DEPLOY.md                 ← This deployment guide
├── .gitignore                ← Git ignore rules
└── README.md                 ← Project overview
```
