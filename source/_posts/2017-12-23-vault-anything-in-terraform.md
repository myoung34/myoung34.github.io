---
title: Vault Roles/Policies in Terraform
author: myoung
layout: post
comments: true
permalink: /post/vault-anything-in-terraform
image:
  -
seo_follow:
  - 'false'
seo_noindex:
  - 'false'
categories:
  - vault
  - aws
  - terraform
tags:
  - vault
  - aws
  - terraform
---

Now we get to the fun part and put everything we need from Vault in terraform. <!-- more -->

If you've used ECS and Vault with the [aws auth backend](https://www.vaultproject.io/docs/auth/aws.html) you may have found a bit of a snag in terms of automation.

Lets say you want to create a new ECS container to run. You want that task to get specific secrets in vault by role. You write the code, terraform, etc and when it runs you get:

```
URL: GET https://server:8200/v1/auth/aws-ec2/role/somerole
Code: 400. Errors:

* entry for role somerole not found
```

So you deployed something but now you have to create a policy, a role, etc in vault via the CLI, all the while your task is throttling.

Little known fact: the terribly-named `vault_generic_secret` from the [provider docs](https://www.terraform.io/docs/providers/vault/index.html) can do create roles/policies in addition to secrets (hence the bad name).

In this part 2 I'll show you how to automate the next step of your ECS task into terraform so there's no race condition.

## Prerequisites

 1. Vault running as a server
 1. Vault CLI to match the server
 1. Terraform

## Create a vault policy in terraform

```
resource "vault_generic_secret" "policy" {
  path = "sys/policy/${aws_iam_role.neutrona_wand.name}"

  data_json = <<EOT
{
  "name": "${aws_iam_role.neutrona_wand.name}",
  "rules": "path \"secret/ops/*\" {\n  policy = \"read\"\n}\n\npath \"secret/common/*\" {\n  policy = \"read\"\n}"
}

EOT
}
```

## Create an AWS auth role in terraform

```
provider "vault" {}

resource "aws_iam_role" "neutrona_wand" {
  name = "neutrona_wand"
  path = "/lambda/"

  assume_role_policy = <<POLICY
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "Service": [
          "lambda.amazonaws.com"
        ]
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
POLICY
}

# The ugly replace in 'bound_iam_principal_arn' is to account for removing the path from the role. 
# If you look, my IAM role is /lambda/, so the arn would be 'arn:aws:iam::1234567890:role/lambda/neutrona_wand'
# While the 'bound_iam_principal_arn' wants the non-path version: 'arn:aws:iam::1234567890:role/neutrona_wand'
resource "vault_generic_secret" "role" {
  path = "auth/aws/role/${aws_iam_role.neutrona_wand.name}"

  data_json = <<EOT
{
  "bound_iam_principal_arn": "${replace(aws_iam_role.neutrona_wand.arn, "/:role\\/[a-zA-Z\\/]*\\//", ":role\\/")}",
  "policies": "default,${aws_iam_role.neutrona_wand.name},slack",
  "max_ttl":"300",
  "resolve_aws_unique_ids": "false"
}
EOT
}
```
