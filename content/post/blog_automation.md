---
title: "Blog Automation"
date: 2026-06-26T10:53:55-05:00
draft: false
description: "How to automate the updating of the Blog files"
categories: ["Tutorial"]
tags: ["aws", "hugo" ]
---

While it was fairly easy to manually copy the files for the blog up to my S3 bucket and then create an invalidation in Cloudfront, I decided that I wanted to streamline that process and see if I could make it even easier.  I want to automate it all using the AWS cli and write a BASH script that will do the deploy.  In the future I might move towards Github actions when I push an update to Github it will do the update, but baby steps.

## Install AWS cli

To automate the AWS things, I needed the AWS cli tools installed on my computer.  That is easy enough using [Homebrew](https://brew.sh), which I highly recommend if using a Mac.

`brew install awscli` 

## User Setup

For my personal AWS account, I have always loged in as the root user.  I know it is not best practice, but since I am the only one doing anything in the account, I figured it was ok.  I decided that since I need programatical access to AWS that it was time I setup some real IAM credentials.  I wanted to make sure I did this with the concept of least priveldges in mind, so for the account, I simply created an account with console access and then setup MFA on that account.  AWS IAM permissions are out of scope for this post, but to allow the user to assign MFA to their account they need the permission IAMReadOnlyAccess added.  

I needed to grant the user rights to be able to delete files from my S3 bucket, copy files to the S3 bucket, and to create an invalidation on my CloudFront distribution as well as sign into the cli tools.  To grant access from the cli tool I created a group and assigned the AWS Permission SignInLocalDevelopmentAccess to the group.  To grant the other permissions, I created an inline policy to the group and added the following policy.

``` json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListBlogBucket",
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:GetBucketLocation"
      ],
      "Resource": "arn:aws:s3:::BUCKET_NAME"
    },
    {
      "Sid": "WriteAndDeleteBlogObjects",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::BUCKET_NAME/*"
    },
    {
      "Sid": "InvalidateCloudFront",
      "Effect": "Allow",
      "Action": "cloudfront:CreateInvalidation",
      "Resource": "arn:aws:cloudfront::AWS_ACCOUNT_ID:distribution/CLOUDFRONT_DISTRIBUTION_ID"
    }
  ]
}
```

Now to be able to use the AWS cli commands and run the script, we need to be logged into AWS to perform those actions.  This can be achieved by setting up an AWS Key ID and secret key that we can use.  But there is a simpler way, the AWS cli tools allow you to log in via the command line, by simply running the command.  
`aws login`  
This will use the web browser to sign you in and will be active for 24 hours.

## Deploy Script

To actually deploy the script I wrote a simple BASH script that will build the [Hugo](https://gohugo.io) files into the public folder, sync the files into the S3 bucket and create the CloudFront invalidation.The invalidation essentially tells CloudFront to get rid of cached files and look again at the S3 bucket.

Run touch to create file.  
`touch deploy.sh`

The script looks like this.

``` bash
#!/bin/bash
set -e

BUCKET="BUCKET_NAME"
DISTRIBUTION_ID="CLOUDFRONT_DISTRIBUTION_ID"

echo "Building Hugo site..."
hugo --gc --minify

echo "Syncing to S3..."
aws s3 sync public/ s3://$BUCKET --delete

echo "Invalidating CloudFront..."
aws cloudfront create-invalidation \
  --distribution-id $DISTRIBUTION_ID \
  --paths "/*"

echo "Deploy complete."
```

Need to make the script executable by running the chmod command.  
`chmod +x deploy.sh`

## Running the script

To reun the script and deply the site you simply run the command.  
`./deploy.sh`  

That's it and now the deploy only takes a couple of seconds to run and reduces any chances of me manually screwing something up.


