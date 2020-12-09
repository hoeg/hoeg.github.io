---
layout: post
title:  "AWS Auth with Hashicorp Vault - HowTo"
date:   2016-07-19 11:49:45 +0200
categories: AWS Vault authentication
---
All of our credentials are stored on Vault. We need an easy way to access these in our AWS environment. Vault provides the AWS Auth authentication method where we will use the IAM type since we are working with roles in our environment. Setting this up should be straightforward but going through the documentation we ran into a lot of unknowns and barriers. This might be due to our limited knowledge of the Vault terminology and concepts, however we feel like this warranted an other perspective on this setup procedure. Hence this post.

The first thing we did was look at Vault own documentation(https://www.vaultproject.io/docs/auth/aws). This talks about the different authentication flows and what the client must and actually also which permissions the Vault instance needs in order to enable AWS Auth.

## Prerequisits

Cross account authentication is not needed at the moment so using terraform we add the policy to our EC2 role instance and apply these changes:

`
resource "aws_iam_role_policy_attachment" "aws_auth_permissions" {
  role       = module.vault_cluster.iam_role_name
  policy_arn = aws_iam_policy.aws_auth_permissions.arn
}

resource "aws_iam_policy" "aws_auth_permissions" {
  name        = "aws_auth"
  description = "Permissions needed to verify AWS roles authenticating to Vault."
  policy      = data.aws_iam_policy_document.aws_auth.json
}

data "aws_iam_policy_document" "aws_auth" {
  statement {
    sid = "VaultAWSAuth"

    actions = [
      "iam:GetUser",
      "iam:GetRole",
    ]

    resources = [
      "arn:aws:iam:::*"
    ]
  }
}
`

## Vault setup

First we set our vault location and then we login:

`# export VAULT_ADDR=…
  # vault login -method=github
`

Now we enable the AWS Auth method:

`vault auth enable aws`

Write policy:

Vault policy write “

vault write auth/aws/role/TestRole auth_type=iam bound_iam_principal_arn="arn:aws:iam::<redacted>:role/vaultEC2Role" -policies="my-policy" mac_ttl=1h
Error writing data to auth/aws/role/MyRole: Error making API request.

URL: PUT http://localhost/v1/auth/aws/role/MyRole
Code: 400. Errors:

* unable to resolve ARN "arn:aws:iam::<redacted>:role/vaultEC2Role" to internal ID: AccessDenied: User: arn:aws:sts::<redacted>:assumed-role/vaultEC2Role/i-0d0159ebebf505f5f is not authorized to perform: iam:GetRole on resource: role vaultEC2Role
	status code: 403, request id: d7347ab1-47ae-4eaa-a11d-1a9172eaa70d

vault auth enable aws
vault write auth/aws/role/<client role> auth_type=iam bound_iam_principal_arn="<client role arn>" policies="<policy" mac_ttl=1

For cross account access add a role (vault_auth) in the client account with the same permissions as 
aws_iam_policy_document.aws_auth and set a trust policy to AWS: arn of vault server role

next add the vault_auth role as the target for vault to use for authentication:
vault write auth/aws/config/sts/<client account> sts_role=<vault auth arn>


$ vault write auth/aws/config/sts/<target account #> sts_role=arn:aws:iam::<target account #> :role/vault_auth                                                                                                                                   2 ↵  10260  10:19:57
Success! Data written to: auth/aws/config/sts/<target account #> 
$ vault write auth/aws/role/MyRole auth_type=iam bound_iam_principal_arn="arn:aws:iam::<target account #>:role/EC2Role" policies="my-policy" mac_ttl=1h
