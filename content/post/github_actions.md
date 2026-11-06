---
title: "Github Actions"
date: 2026-11-06T14:40:04-06:00
draft: false
description: "Details on switching the build and deploy process to Github actions"
categories: ["Tutorial"]
tags: ["Github", "AWS"]
---

The process of running the script to update my blog made the process so much easier, faster and honestly, because I don't post as often I didn't have to remember the steps required to use [Hugo](https://gohugo.io) to build the site and then deploy it on AWS.  I had mentioned in my previous post about deploying it via the script that I might eventually move the process to Github Actions.  This blog details how I achieved that migration.

To be fully transparent about the process, I got a lot of help from [Claude](https://claude.ai) in the creation of [Github](https://github.com) Action as well as the AWS Roles and Permissions needed to complete the Github Action.

## Why bother

The script works fine, so this is not about fixing something that is broken.  I wanted to learn more about CI/CD workflows and Github Actions specifically, and my blog is a pretty safe place to experiment.  As a bonus, I will no longer need to be logged into AWS from my laptop to publish a post, I just push to Github and it does the rest.

My first question was whether I needed to find an action in the marketplace to copy files to S3 and do the CloudFront invalidation.  It turns out I didn't.  The AWS cli is already installed on the Github runners, so the commands are the exact same ones from my deploy script.  The only marketplace action I used is `aws-actions/configure-aws-credentials`, which is made by AWS and handles the login.  I stayed away from the third party S3 sync actions since they are just wrappers around `aws s3 sync` and I would rather not run someone else's code with access to my AWS account.

## How Github logs into AWS

The easy way to do this is to create an IAM user, generate an access key and secret key and store them as secrets in Github.  The problem is those keys never expire unless I remember to rotate them, and if they leak, that is a real problem.

The better way is OpenID Connect, or OIDC.  Github creates a short lived token for each run of the workflow that basically says "I am this repository running this workflow".  AWS is set up to trust those tokens and trades them for temporary credentials.  There is nothing long lived to steal.

To make that work you need three things in AWS, an identity provider, an IAM role and a policy that says what the role is allowed to do.

## Create the identity provider

This is a one time setup for the AWS account.  In the IAM console, go to Identity providers and add a provider using OpenID Connect, with the provider URL `https://token.actions.githubusercontent.com` and the audience `sts.amazonaws.com`.  Or if you prefer the cli.

``` bash
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com
```

Some of the older guides ask for a thumbprint, but AWS now validates the certificate on its own so you can ignore that.  The provider doesn't give anyone any permissions, it only tells AWS that Github's tokens can be trusted.  If it already exists you will get an error saying so, which is fine.

## Create the role

The role has two parts, a trust policy that says who is allowed to use the role and a permissions policy that says what the role can do.  Here is the trust policy, saved as trust-policy.json.

``` json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::AWS_ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:GITHUB_USER/REPO_NAME:environment:production"
        }
      }
    }
  ]
}
```

The most important line is the `sub` condition.  That is what limits the role to my one repository.  If you leave that out, or use a wildcard for the repository, any repository on Github could try to use your role.  The `environment:production` part matches the Github environment that I set on the deploy job in the workflow, which I will get to in a minute.

The permissions policy is almost exactly the same one I used for the IAM user in my previous post, saved as permissions-policy.json.  It only allows what the deploy needs.

``` json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListBlogBucket",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::BUCKET_NAME"
    },
    {
      "Sid": "WriteAndDeleteBlogObjects",
      "Effect": "Allow",
      "Action": [
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

Then create the role and attach the policy using the cli.

``` bash
aws iam create-role \
  --role-name github-actions-blog-deploy \
  --assume-role-policy-document file://trust-policy.json

aws iam put-role-policy \
  --role-name github-actions-blog-deploy \
  --policy-name blog-deploy \
  --policy-document file://permissions-policy.json

aws iam get-role --role-name github-actions-blog-deploy \
  --query Role.Arn --output text
```

That last command prints the ARN of the role, which we need next.

## Tell Github about the role

In the repository on Github, go to Settings, then Secrets and variables, then Actions.  I added a secret named `AWS_DEPLOY_ROLE_ARN` with the ARN of the role and a variable named `AWS_REGION` with the region my bucket is in.  The ARN isn't really sensitive, but it keeps account details out of the workflow file.

## The workflow

Github looks for workflows in the `.github/workflows` folder of the repository, so I created a file there called deploy.yml.

``` yaml
name: Build and deploy blog

on:
  pull_request:
  push:
    branches: [master]
  workflow_dispatch:

permissions:
  id-token: write
  contents: read

concurrency:
  group: deploy
  cancel-in-progress: false

env:
  HUGO_VERSION: x.y.z
  BUCKET: BUCKET_NAME
  DISTRIBUTION_ID: CLOUDFRONT_DISTRIBUTION_ID

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: true

      - name: Install Hugo
        run: |
          curl -sSL "https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.tar.gz" \
            | tar -xz hugo
          sudo mv hugo /usr/local/bin/hugo
          hugo version

      - name: Build
        run: hugo --gc --minify

      - uses: actions/upload-artifact@v4
        with:
          name: public
          path: public/

  deploy:
    needs: build
    if: github.ref == 'refs/heads/master' && github.event_name != 'pull_request'
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: public
          path: public/

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_DEPLOY_ROLE_ARN }}
          aws-region: ${{ vars.AWS_REGION }}

      - name: Sync to S3
        run: aws s3 sync public/ "s3://${BUCKET}" --delete

      - name: Invalidate CloudFront
        run: |
          aws cloudfront create-invalidation \
            --distribution-id "${DISTRIBUTION_ID}" \
            --paths "/*"
```

There are a few things in here that I learned along the way.

The workflow runs in a few different situations.  A pull request only runs the build job, so I can make sure a post builds before I merge it.  A push to master builds and deploys.  The `workflow_dispatch` line adds a Run workflow button in the Actions tab if I ever want to kick it off by hand.

My themes are git submodules and the checkout step doesn't download those by default.  Without `submodules: true` the build fails because Hugo can't find the theme.

Each job runs on its own fresh machine, so the public folder that the build job creates has to be uploaded as an artifact and then downloaded again in the deploy job.  It also means the job that talks to AWS never runs on a pull request.

The `id-token: write` permission is what allows the job to ask Github for the OIDC token, without it the AWS login step fails.  The `environment: production` line is what puts `environment:production` in that token, and it matches the `sub` condition in the trust policy.  It also gives me a place to add an approval step later if I ever want one.

Finally, I downloaded a specific version of Hugo instead of using a marketplace action, so the build matches the version I run on my Mac.

## When it doesn't work

If the deploy fails with an error saying you are not authorized to perform `sts:AssumeRoleWithWebIdentity`, the problem is almost always in the trust policy.  Check the repository name, the `sub` value and the ARN of the provider.  The `sub` needs to match exactly what Github sends.

If the S3 sync fails with access denied, look at the permissions policy.  The `s3:ListBucket` action goes on the bucket and the other actions go on the bucket with `/*` at the end.  It is easy to get those backwards.

## Wrapping up

I left the deploy.sh script in the repository as a backup, but from here on I should only need to push my changes to master.  Some things I want to look at next are adding cache headers to the static files so I am not invalidating everything on every post, and only running the workflow when the content actually changes.  The part I found the most interesting was OIDC and how it removes the need for stored keys, and that is something I will definitely use again.
