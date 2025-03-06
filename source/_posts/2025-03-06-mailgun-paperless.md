---
title: Automating email attachments to paperless-ngx
author: myoung
layout: post
comments: true
permalink: /post/automating-email-attachments-to-paperless-ngx
seo_follow:
  - 'false'
seo_noindex:
  - 'false'
categories:
  - Paperless
  - Automation
tags:
  - Paperless
  - Automation
---

I recently set up paperless-ngx and wanted an easy way to simply email myself receipts or documents and get them uploaded and tagged. <!-- more -->

[paperless-ngx](https://docs.paperless-ngx.com/) is a super cool way to host docs. I wanted originally to just organize and digitize my receipts and taxes in a way that will stay backed up and easy to find. 

Then I realized I want to simplify it because that process can take a long time manually.

First: I set up mailgun for my tld, lets say `mail.foo.com`
Mailgun has a free tier thats more than enough for my low volume use-case, and they allow you to forward messages after filters to a webhook.

I created a route that checks that I sent it and forwards it to something I deployed (will go over that futher below) behind a tailscale funnel (so that it can be exposed to the wider net).


{% img /images/mailgun.png %}

Next I wrote a super minimal flask app called [mailgun-to-paperless-ngx](https://github.com/myoung34/mailgun-to-paperless-ngx) that will take the attachments from mailgun and upload them to paperless-ngx using the `To` as tags. Example: `receipt+work@mail.foo.com` with an attachment `dinner.jpg` will upload it to paperless as `dinner` (title) and tags `receipt` and `work`.

Lets see it by sending this image out to `testing@mail.foo.com`.

{% img /images/gmail.png %}

We see mailgun posted it.

{% img /images/mailgun1.png %}

We also now see that the flask app is processing and uploading it:

```
file saved to /tmp/dinner.jpg
Uploading dinner.jpg to http://paperless-ngx.paperless-ngx.svc.cluster.local:8000 with tags [20]
Upload complete. Task ID: d8148db1-19a2-4392-8e3b-17f16fc905ef
Checking on task d8148db1-19a2-4392-8e3b-17f16fc905ef ...
waiting for task d8148db1-19a2-4392-8e3b-17f16fc905ef to complete...
Checking on task d8148db1-19a2-4392-8e3b-17f16fc905ef ...
task d8148db1-19a2-4392-8e3b-17f16fc905ef completed successfully
```

And now it's in paperless as expected

{% img /images/paperless.png %}
