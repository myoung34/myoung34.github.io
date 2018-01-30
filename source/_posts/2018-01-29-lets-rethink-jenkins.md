---
title: Let's Rethink Jenkins
author: myoung
layout: post
comments: true
permalink: /post/lets-rethink-jenkins
image:
  -
seo_follow:
  - 'false'
seo_noindex:
  - 'false'
categories:
  - jenkins
  - ci
  - elk
tags:
  - jenkins
  - ci
  - elk
---

I lose all data from jenkins every time I deploy it. And that's OK.  <!-- more -->

## Background: a rant
Let me start off by saying: I've done *a lot* of Jenkins. 

 * Exhibit A (super old jenkins with fake pipelines): {% img right /images/jenkins-old.png jenkins 1.4 %}

I've done new Jenkins (I love 2.0 and love Jenkinsfile's).

I've worked with [large jenkins installs](http://jenkins.ovirt.org).
I've worked with small jenkins at multiple shops.

I've automated backups with S3 tarballs:

```
backup_dir=/var/lib/jenkins/backup
if [[ -d $backup_dir ]]; then
  rm -rf $backup_dir/*.tar.gz
else
  mkdir -p $backup_dir
fi
archive_name=jenkins-backup-$(date +%Y-%m-%d-%H_%M_%S).tar.gz
filename=$backup_dir/$archive_name
tar --exclude=backup --exclude=backups -czf $filename --warning=no-file-changed /var/lib/jenkins
aws s3 cp $filename s3://$backup_bucket/backups/$backup_name
```

I've automated backups with [thin backup](https://plugins.jenkins.io/thinBackup).

I've automated creating jobs with [job-dsl](https://github.com/jenkinsci/job-dsl-plugin)

I've automated creating jobs with straight XML.

I've automated creating jobs with [jenkins job builder](https://docs.openstack.org/infra/jenkins-job-builder/).

They all suck. I'm sorry. But they're complex and require a lot of orchestration and customization to work.

## Im gonna do worse

My problem with all these previous methods is they require a hybrid suck. You have to automate the jobs and mix that with retaining your backups. 

*I'm tired of things using the filesystem as a database.*

These things are complex because the data lives in the same place as the jobs. When you build up your jobs they contain metadata about build history. So you end up with these nasty complicated methods of pulling down and merging the filesystem.

My hypothesis: #(@$ that. Lets use a real database and artifact store for the data and just not care. Crazy Right?!

I've [got some demo code](https://github.com/myoung34/docker-jenkins) to prove to you how easy it is. It's 100% groovy. No job dsl, no nothing. Just groovy and java methods.
The one I posted shows how I automated groovy to set up:

 1. [Active directory](https://github.com/myoung34/docker-jenkins/blob/master/ad.groovy)
 1. [AWS ECS agents](https://github.com/myoung34/docker-jenkins/blob/master/ecs.groovy)
 1. [Github organizations (includes webhooks!)](https://github.com/myoung34/docker-jenkins/blob/master/github.groovy)
 1. [Slack](https://github.com/myoung34/docker-jenkins/blob/master/slack.groovy)
 1. [Libraries like better slack messages and jobs defined in a java properties file](https://github.com/myoung34/docker-jenkins/blob/master/jenkins.properties)
 1. [Logstash for the real meat of this post](https://github.com/myoung34/docker-jenkins/blob/master/logstash.groovy)

## The real reason youre here

My jenkins master is docker. Its volatile. And its stateless.

So how do I do it? In production my docker hosts listen to UDP port 555 with Logstash for syslog and forward them to ELK (because SSL, and other reasons).
[In my example the logstash plugin just sends directly to elasticsearch](https://github.com/myoung34/docker-jenkins/blob/master/jenkins/logstash.groovy).
All the env vars are pulled from [vault](http://www.vaultproject.io) at run time and I get to manage the data like my other data from ELK.

My artifacts go to someting like [artifactory](http://www.jfrog.io).

And guess what? It's fantastic. Automation is dead simple. And I put the data in a real database. which lets me do things like see build times across projects, see deployments, etc.

{% img left /images/jenkins1.png jenkins %}
{% img left /images/jenkins3.png jenkins %}
