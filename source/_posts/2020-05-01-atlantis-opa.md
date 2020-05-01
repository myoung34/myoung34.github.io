---
title: Terraform and Open Policy Agent with Atlantis
author: myoung
layout: post
comments: true
permalink: /post/atlantis-opa
image:
  -
seo_follow:
  - 'false'
seo_noindex:
  - 'false'
categories:
  - opa
  - conftest 
  - terraform
  - atlantis
tags:
  - opa
  - conftest 
  - terraform
  - atlantis
---

At Zapier we really like Atlantis and smart pre-flight checks. <!-- more -->

If you see a trend in my latest blogs it's likely you'll guess i really like [OPA](https://www.openpolicyagent.org). 

We use it in EKS to report on assertions consistently across all our clusters.

I use it to run tests against the gatekeeper rules.

It's pretty awesome, and it [supports terraform plans](https://www.openpolicyagent.org/docs/latest/terraform).

### The Set-Up ###

I started off by forking the upstream atlantis AMI and baking the aws-cli into it.

In my atlantis.yaml I set up all the repos to use a new workflow `opa-workflow` such:

```
workflows:
  opa-workflow:
    plan:
      steps:
        - init
        - plan
        - run: terraform$ATLANTIS_TERRAFORM_VERSION workspace select -no-color $WORKSPACE
        - run: terraform$ATLANTIS_TERRAFORM_VERSION show -json $PLANFILE >"${PROJECT_NAME}_${PULL_NUM}_$(git rev-parse HEAD).json"
        - run: aws s3 cp "${DIR}/${PROJECT_NAME}_${PULL_NUM}_$(git rev-parse HEAD).json" s3://bucket_name/terraform/tfplan/ >/dev/null
```

This pushes the plan file as `production_1234_somesha11122.json` into the bucket.

Next we have an AWS lambda with conftest built in. It responds to `terraform/tfplan/*.json` S3 create notifications, parses out the SHA, PR number, and workspace from atlantis, then runs:

```
command:= exec.Command(
  "/var/task/bin/conftest",
  "test",
  "--no-color",
  "--output",
  "json",
  "/tmp/tfplan.json"
)
output, err := command.CombinedOutput()
if err != nil {
  fmt.Printf("[%s] conftest failed: %v", time.Now().Format("2006-01-02 15:04:05"), err)
}
```

Once this works, we can parse back the info to a comment and send a success|failure status to the git sha!

{% img /images/terropa.png %}
