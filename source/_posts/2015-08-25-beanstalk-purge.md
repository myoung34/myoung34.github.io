---
title: Purging old beanstalk versions
author: myoung
layout: post
comments: true
permalink: /post/beanstalk-purge
image:
  - 
seo_follow:
  - 'false'
seo_noindex:
  - 'false'
categories:
  - aws
  - beanstalk
  - ruby
tags:
  - aws
  - beanstalk
  - ruby
---

I recently found out that you can have at most 500 application versions across beanstalk. I wrote a small script to purge it of old ones<!-- more -->

I'm a big fan of micro-deployments, however that seems to have caused a headache lately (and it turns out you don't want to find this out when you absolutely need to deploy).

My first attempt was to just browse to the S3 folder for our applications and purge them from S3 by the oldest ones, but that could cause a problem if you delete the archive for a currently deployed version. Also, as it turns out, Beanstalk caches this object list, so you still have to go to the EB console and click 'Refresh' in the application versions list.

The better solution was to write a small ruby snippet to run on cron. It will get all application versions, subtract currently deployed ones, sort by date created, and purge all except 'x' many.

{% highlight ruby %}
require 'aws-sdk'

MAX_VERSIONS=50
elasticbeanstalk = Aws::ElasticBeanstalk::Client.new(region: 'us-east-1')

short_versions_to_keep = elasticbeanstalk.describe_environments.environments.each.map { |v| "#{v.application_name}+#{v.version_label}" }

version_eligible_for_delete = {}
elasticbeanstalk.describe_application_versions.application_versions.each do |av|
  short_version = "#{av.application_name}+#{av.version_label}"
  next if short_versions_to_keep.include? short_version
  version_eligible_for_delete[av.application_name] ||= {}
  version_eligible_for_delete[av.application_name][av.version_label] = av
end

version_eligible_for_delete.each do |key, value|
  version_eligible_for_delete[key] = value.sort { |a,b| a[1].date_created <=> b[1].date_created }
  if version_eligible_for_delete[key].size <=MAX_VERSIONS
    puts "#{key} has only #{version_eligible_for_delete[key].size} versions. Skipping"
    next
  end
  version_eligible_for_delete[key][0..version_eligible_for_delete[key].size-MAX_VERSIONS].each do |a|
    puts "Deleting #{a[1].application_name}: #{a[1].version_label}"
    elasticbeanstalk.delete_application_version(application_name: a[1].application_name, version_label: a[1].version_label)
    sleep 1
  end
end

{% endhighlight %}
