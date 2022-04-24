# Static Web Hosting On AWS with S3

*A guide by and for Grant*

## Introduction



This guide is a down-and-dirty guide for how to host static web content through AWS with a custom domain and SSL certificate. 

# Assumptions

1. You already have your domain in Route 53
2. For this example, I will go through with example.com

## Step 1 - Request Certificate

1. In certificate manager request the certificate for example.com, leave it on DNS validation
2. After the certificate has been created, wait until the validation record names and values are loaded. Then click create records. 

## Step 2 - Create and Configure S3 Bucket

1. Create the S3 bucket with the name example.com
2. Untick the checkmark called "block all public access"
3. Scroll down and click create
4. Upload your content
5. Go to the new bucket called example.com, click on the permissions tab, then click edit under Bucket policy and add the following configuration

``` {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "Statement1",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::example.com/*"
        }
    ]
}
```

## Step 3 - Create CloudFront Distribution

1. Go to CloudFront, click create distribution

   1. Origin name should be the bucket you just configured
   2. Under viewer click "Redirect HTTP to HTTPS"
   3. Under Settings, click add item under "Alternate Domain name (CNAME) - Optional", write example.com
   4. Under Settings, click the drop-down under "Custom SSL - Optional", choose the Certificate you made earlier
   5. Under Settings, write index.html under "Default root object *- optional*"
   6. Click Create distribution

   *NOTE: This will take ~5 minutes to deploy, wait for this to happen*

2. Go to Route 53, create a new reccord in your hosted zone for example.com

   1. leave the record name blank
   2. Select an A record
   3. Mark it as an alias
   4. Under "Route Traffic to" Select "Alias to CloudFront distribution"
   5. Select the distribution you created earlier
   6. Click create record

   *NOTE: You will need to wait a few minutes for this to propagate and clear your cache to see this change.*