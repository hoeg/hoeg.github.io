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



TF_LOG=DEBUG 
