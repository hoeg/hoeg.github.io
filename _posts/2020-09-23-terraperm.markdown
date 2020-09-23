---
layout: post
title:  "AWS least privilege for terraform deployment"
date:   2020-09-23 11:49:45 +0200
categories: IaC
---

I have been working with terraform and AWS for a couple of years and often times when I 

One of the best practices of using AWS it to employ the principle of least privilege when creating policies for roles and users [1]. But how do we ensure that the policy used for applying our terraform has the correct set of privileges and no more than it needs? We could try to start with a full Deny policy and do the following workflow:

1. apply terraform
2. find out which permission we don't have
3. add said permission to policy
4. back to step one if not success

It must be possible to do better than this. Having done this so many times I decided to automate this task. If we could somehow record all the API calls made by terraform to AWS and from those deduce a policy only allowing these calls, we would be done. Fortunately for us terraform makes it possible to turn on debug log by setting the environment variable

`TF_LOG=DEBUG`[2].

Running `terraform apply` with this variable set will write a detailed log from terraform and to our luck this log contains all the AWS API calls.

```
...
2020-09-20T21:36:31.528+0200 [DEBUG] plugin.terraform-provider-aws_v3.7.0_x5: 2020/09/20 21:36:31 [DEBUG] [aws-sdk-go] DEBUG: Request s3/CreateBucket Details:
2020-09-20T21:36:31.528+0200 [DEBUG] plugin.terraform-provider-aws_v3.7.0_x5: ---[ REQUEST POST-SIGN ]-----------------------------
2020-09-20T21:36:31.528+0200 [DEBUG] plugin.terraform-provider-aws_v3.7.0_x5: PUT / HTTP/1.1
2020-09-20T21:36:31.528+0200 [DEBUG] plugin.terraform-provider-aws_v3.7.0_x5: Host: hoeg-terraperm-bucket.s3.eu-west-1.amazonaws.com
2020-09-20T21:36:31.528+0200 [DEBUG] plugin.terraform-provider-aws_v3.7.0_x5: User-Agent: aws-sdk-go/1.34.21 (go1.14.5; linux; amd64) APN/1.0 HashiCorp/1.0 Terraform/0.13.2 (+https://www.terraform.io)
2020-09-20T21:36:31.528+0200 [DEBUG] plugin.terraform-provider-aws_v3.7.0_x5: Content-Length: 153
2020-09-20T21:36:31.528+0200 [DEBUG] plugin.terraform-provider-aws_v3.7.0_x5: Authorization: AWS4-HMAC-SHA256 Credential=AKIA4TV6MKMV6ZNQRJNN/20200920/eu-west-1/s3/aws4_request, SignedHeaders=content-length;host;x-amz-acl;x-amz-content-sha256;x-amz-date, Signature=134fd1bcb305ac797f652c2cd7a0987797ff45aa2e6e06c44ba15c4b6d998f4b
2020-09-20T21:36:31.528+0200 [DEBUG] plugin.terraform-provider-aws_v3.7.0_x5: X-Amz-Acl: private
2020-09-20T21:36:31.528+0200 [DEBUG] plugin.terraform-provider-aws_v3.7.0_x5: X-Amz-Content-Sha256: a2531158b25edb57200e54140cc1b12d0af943f596ea467c2fbbedcc16223cc5
2020-09-20T21:36:31.528+0200 [DEBUG] plugin.terraform-provider-aws_v3.7.0_x5: X-Amz-Date: 20200920T193631Z
2020-09-20T21:36:31.528+0200 [DEBUG] plugin.terraform-provider-aws_v3.7.0_x5: Accept-Encoding: gzip
2020-09-20T21:36:31.528+0200 [DEBUG] plugin.terraform-provider-aws_v3.7.0_x5: 
2020-09-20T21:36:31.528+0200 [DEBUG] plugin.terraform-provider-aws_v3.7.0_x5: <CreateBucketConfiguration xmlns="http://s3.amazonaws.com/doc/2006-03-01/"><LocationConstraint>eu-west-1</LocationConstraint></CreateBucketConfiguration>
...
```
From this request we can deduce the following:

| Type       | Value                               |            |
|---         |---                                  |---         |
| Service    | S3                                  |            |
| Action     | CreateBucket                        |            |
| Arn        | arn:aws:s3:::hoeg-terraperm-bucket  |            |
| Condition  | aws:RequestedRegion                 | eu-west-1  |

we will have to parse each request to get the required data
this is translated to:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Stmt1600888858900",
      "Effect": "Allow",
      "Action": [
        "s3:CreateBucket
      ],
      "Resource": "arn:aws:s3:::hoeg-terraperm-bucket"
      "Condition": {
        "ArnEquals": {
          "aws:RequestedRegion": "eu-west-1"
        }
      }
    }
  ]
}
```

We will do this translation for all the calls to the AWS API. We will be left with at statement for each API call which is not optimal since  

and output a policy with all the permissions needed in order to apply the terraform.

