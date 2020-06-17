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

We really like Atlantis and smart pre-flight checks. <!-- more -->

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

```
if statusString == "failure" {
  atlantisStatus := &github.RepoStatus{
    State: github.String(statusString),
    Description: github.String("TerrOPA Failures"),
    Context: github.String("atlantis/plan"),
  }
  client.Repositories.CreateStatus(backgroundctx, orgName, repoName, gitSha, atlantisStatus)
}
```

{% img /images/terropa.png %}


## Update ##

I've had a few people ask me why I did it this way as an AWS lambda vs just running it on atlantis. I actually tried this first, but it wasn't "clean".

First: if atlantis plan fails, its not very intuitive, *especially* if it's not a terraform error. It's kind of an unformatted stack trace.

{% img /images/terropa2.png %}

With that exception it's not very feedback friendly. The point of this project is to provide a clean feedback loop for those who may not know or care what OPA is. We have quite a few methods of automatically generating terraform projects from cookie cutter + modules. I wanted a way of providing as much context as possible.

Next: we have a fairly large engineering team and I like to work in a "plugin" modal. When I'm introducing something new I like to be able to toggle it on and off quickly, or gather metrics before pushing for changes. In this way, it's completely separated from Atlantis and has its own SDLC, which is nice from a "Marc, you broke everything" perspective.

Lastly I had to make a call from a security perspective. The point of this is 100% to be able from an upper architectural/infosec level to make assertions. For example: tagging. Tagging for us is partially for billing, partially to see our accountability plane, but *mostly* so that we can scope least privilege. 

Having this with atlantis would mean the policies live in the atlantis repo for the container builder - I dont want to bounce atlantis every time we add/change policies.

Otherwise it would mean the policies live with the terraform which means that you could modify the assertions per-branch which is also a choice we made to avoid. These policies are open for contribution from everyone but it's an explicit rule that it's not meant to be able to shifted per branch and introduce a threat model around modifying it to pass when it shouldn't.
