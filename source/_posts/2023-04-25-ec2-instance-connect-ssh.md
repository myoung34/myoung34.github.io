---
title: Just In Time SSH Using EC2 Instance Connect
author: myoung
layout: post
comments: true
permalink: /post/ec2-instance-connect-ssh
seo_follow:
  - 'false'
seo_noindex:
  - 'false'
categories:
  - AWS
  - SSH
tags:
  - AWS
  - SSH
---

I played around with just in time users in EC2 using instance connect and a custom built NSS module <!-- more -->

I needed to PoC a solution for SSH users that are backed by an IdP but ran into some issues.

EC2 Instance Connect (IMO) seems to be way under promoted by AWS, and I feel like that's a real disservice if you need (I'm so sorry for you) SSH (I do for this case).

The TLDR for ec2 instance connect:

* A user makes a call `aws ec2-instance-connect send-ssh-public-key --instance-id i-08e2c277f1ea9a99a --instance-os-user marcus.young --ssh-public-key file:///home/myoung/.ssh/asdf.pub`
* AWS pushes the public key to `http://169.254.169.254/latest/meta-data/managed-ssh-keys/active-keys/marcus.young` for `60` seconds
* The user does `ssh -i {private key that matches the public key} marcus.young@{public ip of the instance}` and gets in. 

That's crazy cool, but we have a few issues outright (major ones):

* You can impersonate. There's nothing keeping me from doing `--instance-os-user not.me` and getting in. Fine in small cases bad for observability. You could tie this back to a user but that sucks. Youd have to tie back the `SendSSHPublicKey` cloudtrail call with the iam user/role via the request params to the box, and if youre not shipping auth.log youre screwed. If 100 people do this at once you've lost. There's no way to see which key material let who in to run what. If you thought "Oh I'll use sts principaltags" I love you. Keep reading.
* Users have to exist ahead of time. This is the **big** one. If you thought "What if we create them when they send they create the key on the host". You can't. Not easily anyway. If you think it's still solvable or said "What about NSS" keep reading homie.
* There's no documented rate limits. I've brought this up, no answer yet. I might see if I can break this ;)
* You cannot _directly_ do this to private instances. You'll need public instances or bastions to do this. I'll give you a snippet for that too.

Let's get started.

First we need to bring up an Okta with IAM identity center.
Sorry peeps but if you think I'm doing **that** walkthrough just `ctrl+w` and touch grass. Not happening.

Let's assume you have a working IdP and you can get into AWS. Let's start there.

So impersonation....

The answer here is an SCP and [sts principalTags](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_session-tags.html).
For the principal tags, it's sort of straight forward. Unless you discover an IAM identity center bug like I did.
If you're using IAM identity center: You'll need to use 
`https://aws.amazon.com/SAML/Attributes/AccessControl:{ attr name}` 
in Okta.

The bug? Do **not** use both `AccessControl` and `PrincipalTag`. Apparently the IdC freaks out during the SAML jank and wont do anything.

So in Okta, add this:

{% img /images/ec2-ssh/okta1.png %}

Also while you're add it make sure you're using the nickName for the username.

{% img /images/ec2-ssh/okta2.png %}

At this point go ahead and measure success by making sure that your user is in the `first.last` format and your `AssumeRoleWithSAML` call in Cloudtrail has the principalTags on the `requestParameters`. And by that I mean get angry and weep for 8 hours then find out you're hitting an invisible AWS bug. I'll wait.

Thanks for coming back.

Your user doesn't have to be in this format but it tracks with my needs and my SCP as well as my janky NSS code. Modify it if you need. You do you.

{% img /images/ec2-ssh/okta3.png %}

```
    "eventName": "AssumeRoleWithSAML",
    "awsRegion": "us-east-1",
    "sourceIPAddress": "5.3.9.1",
    "userAgent": "aws-internal/3 aws-sdk-java/1.12.451 blahblahblah",
    "requestParameters": {
        "sAMLAssertionID": "_94942c8c-6b73-4912-86f9-14c",
        "roleSessionName": "marcus.young",
        "principalTags": {
            "userName": "marcus.young"
        },
```

OK. Now that you're getting the metadata lets lock it down.

We're going to apply this SCP:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyPublicKeyNotOwnedBySelf",
      "Effect": "Deny",
      "Action": "ec2-instance-connect:SendSSHPublicKey",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "ec2:osuser": "${aws:PrincipalTag/userName}"
        }
      }
    }
  ]
}
```

Very cool right?
What we've done now is said "Any call to `SendSSHPublicKey` has to have a `--instance-os-user` param that matches your `userName` principalTag that was set by okta

Prove it:

```
$ aws ec2-instance-connect send-ssh-public-key \         
    --region us-east-1 \
    --availability-zone us-east-1a \
    --instance-id i-08843505c12cf9d7e \
    --instance-os-user myoung \
    --ssh-public-key file:///home/myoung/.ssh/asdf.pub | jq .

An error occurred (AccessDeniedException) when calling the SendSSHPublicKey operation: User: arn:aws:sts::1111111111:assumed-role/AWSReservedSSO_AdministratorAccess_e3aec0c44fb7c2e0/marcus.young is not authorized to perform: ec2-instance-connect:SendSSHPublicKey on resource: arn:aws:ec2:us-east-1:847713735871:instance/i-08843505c12cf9d7e with an explicit deny in a service control policy
```

```
$ aws ec2-instance-connect send-ssh-public-key \
    --region us-east-1 \
    --availability-zone us-east-1a \
    --instance-id i-08843505c12cf9d7e \
    --instance-os-user marcus.young \
    --ssh-public-key file:///home/myoung/.ssh/asdf.pub | jq .

{
  "RequestId": "5b47d1c2-f8dc-447b-8a0f-6cd5bf060d89",
  "Success": true
}
```

Bam. Now you can't straight up ssh as others outright.

OK that part's done, but we still have another blocker: the user has to exist first. 
That sucks if you have a thousand users. I don't know about you but I don't want to have to sit there and crawl Okta for users that may never ssh in.

OK So the way ec2-instance-connect software works (remember above about what SendSSHPublicKey does and ends it to the metadata endpoint):

* It adds some `AuthorizedKeysCommand` lines to `/etc/ssh/sshd_config`
* When A user tries to SSH in and fails first it will then hit `http://169.254.169.254/latest/meta-data/managed-ssh-keys/active-keys/{user name}`
* SSH Requires that user to exist. If it does, and the public key in memory from the metadata endpoint matches their private key: all good come on in bro

So we can't rely on crawling, we can't rely on ssh, but we **can** rely on [nss](https://man7.org/linux/man-pages/man5/nss.5.html) since thats what ssh relies on for `passwd`.
OK so I sound like an expert but I'm not. A large chunk of this was hacked out by another person at work, but I'm familiar enough. Don't lose hope in me yet.

So SSH talks to NSS. We can hook into NSS to do some logic and create a user like [this](https://github.com/myoung34/ec2-instance-connect-libnss-create)

So now in my user-data I have this sin:

```
#cloud-config
packages:
 - gcc
 - unzip

write_files:
  - path: /usr/local/bin/install-nss-create.sh
    owner: root:root
    permissions: '0777'
    content: |
      #!/bin/bash
      pushd . >/dev/null 2>&1
      cd /tmp
      curl -sL https://github.com/myoung34/ec2-instance-connect-libnss-create/archive/refs/heads/main.zip -o /tmp/nss-create.zip
      unzip nss-create.zip
      cd ec2-instance-connect-libnss-create-main
      make
      make install
      sed -i.bak 's/^passwd:.*sss files$/passwd:     sss files create files/g' /etc/nsswitch.conf
      popd >/dev/null 2>&1

runcmd:
  - [/usr/local/bin/install-nss-create.sh]
```

I basically compile my nss garbage and add `create files` to `passwd` in `/etc/nsswitch.conf`

This means that when a user attempts to SSH it will hook in my C code that calls a script
That script does a 
`curl http://169.254.169.254/latest/meta-data/managed-ssh-keys/active-keys/{user name}`. 

If it gets a 200-300 code (the user has put a public key in there - remember this only works for 60s) then it will create the user.

After `create` I still have `files` so that SSH will go back and look them back up in `/etc/passwd`. So assuming they ran the `aws ec2-instance-connect send-ssh-public-key` command then tried to ssh within 60 seconds: they'll have a user with a matching public key and they'll get in.

Bonus: we should be able to do this to get into private instances.

For this proof I'm going to assume you have a public instance `1.2.3.4` and a private instance you want to reach `10.0.0.50`

Let's make your `~/.ssh/config` file look like this:

```
Host my-bastion
  IdentityFile ~/.ssh/asdf
  Hostname 1.2.3.4
  User marcus.young

Host private-vps
  IdentityFile ~/.ssh/asdf
  ProxyJump my-bastion
  StrictHostKeyChecking no
  Hostname 10.0.0.50
  User marcus.young
  UserKnownHostsFile /dev/null
```

Let's jump to that `private-vps` by using the `my-bastion` as a jump host.
Hint: You'll need to make 2 SendSSHPublicKey calls. One to the public (so you can get into it) and one to the private (so you can also get into it)

First let's prove we can't just get in. So you know I ain't lyin.

```
$ ssh my-bastion 
marcus.young@1.2.3.4: Permission denied (publickey,gssapi-keyex,gssapi-with-mic).

$ ssh private-vps
marcus.young@1.2.3.4: Permission denied (publickey,gssapi-keyex,gssapi-with-mic).
kex_exchange_identification: Connection closed by remote host
Connection closed by UNKNOWN port 65535
```

Told you. Let's push the key to both. (remember you have 60s. obviously id write a tool for this later).

```
$ aws ec2-instance-connect send-ssh-public-key \
    --region us-east-1 \
    --availability-zone us-east-1a \
    --instance-id i-08e2c277f1ea9a99a \
    --instance-os-user marcus.young \
    --ssh-public-key file:///home/myoung/.ssh/asdf.pub | jq .Success
true

$ aws ec2-instance-connect send-ssh-public-key \
    --region us-east-1 \
    --availability-zone us-east-1a \
    --instance-id i-004b083d55a6d76c7 \
    --instance-os-user marcus.young \
    --ssh-public-key file:///home/myoung/.ssh/asdf.pub | jq .Success
true

$ ssh 10.0.0.50
Last login: Tue Apr 25 19:54:31 2023 from ip-10-0-0-50.ec2.internal

   __|  __|  __|
   _|  (   \__ \   Amazon Linux 2 (ECS Optimized)
 ____|\___|____/

For documentation, visit http://aws.amazon.com/documentation/ecs
1 package(s) needed for security, out of 3 available
Run "sudo yum update" to apply all updates.```
```

Very cool. So what did we accomplish? 

* New instances have absolutely zero requirements on anything other than that very early PoC libnss module. Ideally I could package that up.
* We have the normal SSH auditing story. auth.log lines up. We have cloudtrail logs that tell us people are ssh'ing in and where.
* We have JIT users. 
* We use the same IAM stories for ssh. If I have 8 accounts I know that you need access to an AWS account to get to the instances into it. We could go further and add more principal tags to say that you can't do sudo unless you have a tag, or that you can't ssh at all if you don't have a certain department. This  is *leagues* better than some of the paid software I've tested.
* We increased no load to any custom tooling. Teleport has etcd, an SSH CA has some unique problems to solve, etc.
* Our onboarding and offboarding story for ssh is now the same as IAM
* In an incident etc we can simply block SendSSHPublicKey for suspicious users and `pkill` sshd for the named user (a win over shared users) on the bastion and effectively kill them off everywhere.
