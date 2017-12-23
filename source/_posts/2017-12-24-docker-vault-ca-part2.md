---
title: Vault as a CA for ECS containers using Terraform (Part 2)
author: myoung
layout: post
comments: true
permalink: /post/docker-vault-ca-part2
image:
  -
seo_follow:
  - 'false'
seo_noindex:
  - 'false'
categories:
  - docker
  - vault
  - certificates
  - aws
tags:
  - docker
  - vault
  - certificates
  - aws
---

Now for some cool snippets on how to automatically request certs from the Intermediary we created in Part 1<!-- more -->

## Dockerfile

An example of how I typically get Vault into my base Container:

```
FROM debian:jessie
ENV VAULT_VERSION 0.8.1
ENV DUMBINIT_VERSION 1.2.0

RUN apt-get update && apt-get install -y \
		apt-transport-https \
		ca-certificates \
		wget \
		unzip \
		jq \
		curl \
    nginx \
	--no-install-recommends && rm -rf /var/lib/apt/lists/*

RUN curl -L --silent -o vault.zip https://releases.hashicorp.com/vault/${VAULT_VERSION}/vault_${VAULT_VERSION}_linux_amd64.zip && \
  unzip vault.zip && \
  mv vault /usr/local/bin/vault && \
  chmod +x /usr/local/bin/vault

RUN curl -L --silent -o /usr/local/bin/dumb-init https://github.com/Yelp/dumb-init/releases/download/v${DUMBINIT_VERSION}/dumb-init_${DUMBINIT_VERSION}_amd64 && \
  chmod +x /usr/local/bin/dumb-init

RUN mkdir -p /opt/ssl
COPY docker-entrypoint.sh /
RUN chmod +x /docker-entrypoint.sh
ENTRYPOINT ["/usr/local/bin/dumb-init", "--"]
CMD ["/docker-entrypoint.sh"]
```

## The Entrypoint

```
#!/bin/bash

export SSL_PATH=/opt/ssl

EC2_AVAIL_ZONE=$(curl -s --max-time 5 http://169.254.169.254/latest/meta-data/placement/availability-zone)
if [[ $? -eq 0 ]]; then
  # shellcheck disable=SC2001,SC2006
  EC2_REGION="`echo \"$EC2_AVAIL_ZONE\" | sed -e 's:\([0-9][0-9]*\)[a-z]*\$:\\1:'`"
  export AWS_DEFAULT_REGION=$EC2_REGION
  export AWS_REGION=$EC2_REGION
fi

[[ -z "${VAULT_TOKEN}" ]] && vault auth -method=aws >/dev/null

VAULT_JSON="$(vault write -format=json cuddletech_ops/issue/jenkins common_name=jenkins ttl=15551999)"

echo "$VAULT_JSON" | jq -r .data.private_key > ${SSL_PATH}/key.pem
echo "$VAULT_JSON" | jq -r .data.certificate > ${SSL_PATH}/cert.pem
echo "$VAULT_JSON" | jq -r .data.issuing_ca  > ${SSL_PATH}/ca.pem
cat ${SSL_PATH}/cert.pem ${SSL_PATH}/ca.pem  > ${SSL_PATH}/bundle.pem

# run your command and point to the certs generated above. Example in nginx:
#        ssl                  on;
#        ssl_certificate      /opt/ssl/bundle.pem;
#        ssl_certificate_key  /opt/ssl/key.pem;
```

## Terraforming the certificate issuer

If you somehow get the previous part to run, it will fail because you have to create the role in Vault before you can just start generating certs off it. In the [cuddletech guide](http://cuddletech.com/?p=959) this is under `Requesting a Certificate for a Web Server` as:

```
$ vault write cuddletech_ops/roles/web_server \
> key_bits=2048 \
> max_ttl=8760h \
> allow_any_name=true
Success! Data written to: cuddletech_ops/roles/web_server
```

But what if I told you, you could do that in terraform?

```
variable "role" {
}

resource "vault_generic_secret" "pki_role" {
  path = "cuddletech_ops/roles/${var.role}"

  data_json = <<EOT
{
  "key_bits": "2048",
  "max_ttl": "8760h",
  "allow_any_name": "true"
}
EOT
}
```

If you add that with your ECS task along with the vault role and policy from the previous post about Vault Anything in Terraform, all your dependencies are met and kept in code!

You can now generate certificates off your PKI backend and grab secrets all in one go!
