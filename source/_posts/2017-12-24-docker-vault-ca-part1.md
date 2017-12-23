---
title: Vault as a CA for ECS containers using Terraform (Part 1)
author: myoung
layout: post
comments: true
permalink: /post/docker-vault-ca-part1
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

In this I'll show some examples on how to leverage HashiCorp Vault as an awesome CA. Note this part is boring and 80% copy/paste from a cuddletech blog. <!-- more -->

## Prerequisites

 1. Vault running as a server
 1. Vault CLI to match the server

## Create the Root cert/key

(This is actually mostly a clone from [here](http://cuddletech.com/?p=959) but I want to make sure it never disappears)

*Note*: You should not let the vault server generate the root CA, you should generate it and import it. I just havent figured out how to do that yet. I will update this post when I get around to it. [More info on that here](https://en.wikipedia.org/wiki/Offline_root_certificate_authority)

1. Mount the PKI backend for the root

```
$ vault mount -path=cuddletech -description="Cuddletech Root CA" -max-lease-ttl=87600h pki
Successfully mounted 'pki' at 'cuddletech'!
$ vault mounts
Path         Type       Default TTL  Max TTL    Description
cubbyhole/   cubbyhole  n/a          n/a        per-token private secret storage
cuddletech/  pki        system       315360000  Cuddletech Root CA
secret/      generic    system       system     generic secret storage
sys/         system     n/a          n/a        system endpoints used for control, policy and debugging 
```

```
$ vault write cuddletech/root/generate/internal \
> common_name="Cuddletech Root CA" \
> ttl=87600h \
> key_bits=4096 \
> exclude_cn_from_sans=true
Key             Value
---             -----
certificate     -----BEGIN CERTIFICATE-----
MIIFKzCCAxOgAwIBAgIUDXiI3GDzP2IbQ9IatFSCv9Pq/lgwDQYJKoZIhvcNAQEL
BQAwHTEbMBkGA1UEAxMSQ3VkZGxldGVjaCBSb290IENBMB4XDTE2MDcwOTA4MTIz
..
axscmLdVE2HTB87W1H77iKKN8n9Xne//LUidxVX0Kg==
-----END CERTIFICATE-----
expiration      1783411981
issuing_ca      -----BEGIN CERTIFICATE-----
MIIFKzCCAxOgAwIBAgIUDXiI3GDzP2IbQ9IatFSCv9Pq/lgwDQYJKoZIhvcNAQEL
BQAwHTEbMBkGA1UEAxMSQ3VkZGxldGVjaCBSb290IENBMB4XDTE2MDcwOTA4MTIz
...
axscmLdVE2HTB87W1H77iKKN8n9Xne//LUidxVX0Kg==
-----END CERTIFICATE-----
serial_number   0d:78:88:dc:60:f3:3f:62:1b:43:d2:1a:b4:54:82:bf:d3:ea:fe:58
```

Verify

```
$ curl -s http://localhost:8200/v1/cuddletech/ca/pem | openssl x509 -text
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            0d:78:88:dc:60:f3:3f:62:1b:43:d2:1a:b4:54:82:bf:d3:ea:fe:58
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN=Cuddletech Root CA
        Validity
            Not Before: Jul  9 08:12:31 2016 GMT
            Not After : Jul  7 08:13:01 2026 GMT
        Subject: CN=Cuddletech Root CA
...
```

Fix the URL for accessing the CA/CRL URLs

```
$ vault write cuddletech/config/urls issuing_certificates="http://10.0.0.22:8200/v1/cuddletech
Success! Data written to: cuddletech/config/urls
```

## Create the Intermediate cert/key

Mount the PKI for the Intermediate (similar to root)

```
vault mount -path=cuddletech_ops -description="Cuddletech Ops Intermediate CA" -max-lease-ttl=26280h pki
Successfully mounted 'pki' at 'cuddletech_ops'!

$ vault mounts
Path             Type       Default TTL  Max TTL    Description
cubbyhole/       cubbyhole  n/a          n/a        per-token private secret storage
cuddletech/      pki        system       315360000  Cuddletech Root CA
cuddletech_ops/  pki        system       94608000   Cuddletech Ops Intermediate CA
```

Generate the intermediate CSR

```
$ vault write cuddletech_ops/intermediate/generate/internal \
>  common_name="Cuddletech Operations Intermediate CA"
>  ttl=26280h \
>  key_bits=4096 \
>  exclude_cn_from_sans=true
Key     Value
---     -----
csr     -----BEGIN CERTIFICATE REQUEST-----
MIICuDCCAaACAQAwMDEuMCwGA1UEAxMlQ3VkZGxldGVjaCBPcGVyYXRpb25zIElu
dGVybWVkaWF0ZSBDQTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBALt8
...
hD8cpHTXqjKExYWKc/rQDgjw9+RNDdb45xszDagrgFgNPqI9i0fNh9jViMmjUiTc
PQTZS4XxIoRrx1/xVHJ4Qm++ntLPVCvzjMZafg==
-----END CERTIFICATE REQUEST-----
```

Cut and paste that CSR into a new file cuddletech_ops.csr. The reason we output the file here is so we can get it out of one backend and into another and then back out.

```
$ vault write cuddletech/root/sign-intermediate \
>  csr=@cuddletech_ops.csr \
>  common_name="Cuddletech Ops Intermediate CA" \
>  ttl=8760h
Key             Value
---             -----
certificate     -----BEGIN CERTIFICATE-----
MIIEZDCCAkygAwIBAgIUHuIhRF3tYtfoZiAFdjcCtQpMR+cwDQYJKoZIhvcNAQEL
BQAwHTEbMBkGA1UEAxMSQ3VkZGxldGVjaCBSb290IENBMB4XDTE2MDcwOTA4Mjkz
...
UtI2b/AamAqf340eRKmSdEh4WypB4JR+t259YA45w2j4mS+rxREycEk4YosR/vUs
jekMiq57yNq7h8eOTrnOulJxazbVrYGb
-----END CERTIFICATE-----
expiration      1470645002
issuing_ca      -----BEGIN CERTIFICATE-----
MIIFKzCCAxOgAwIBAgIUDXiI3GDzP2IbQ9IatFSCv9Pq/lgwDQYJKoZIhvcNAQEL
BQAwHTEbMBkGA1UEAxMSQ3VkZGxldGVjaCBSb290IENBMB4XDTE2MDcwOTA4MTIz
..
1FRGlwHUg+6IIZBVIapzivLc6pAvLFPxQlQvT5CNHPk91zwyNQ9ZX2PzatdajUnd
axscmLdVE2HTB87W1H77iKKN8n9Xne//LUidxVX0Kg==
-----END CERTIFICATE-----
serial_number   1e:e2:21:44:5d:ed:62:d7:e8:66:20:05:76:37:02:b5:0a:4c:47:e7
```

Now that we have a Root CA signed cert, we’ll need to cut-n-paste this certificate into a file we’ll name cuddletech_ops.crt and then import it into our Intermediate CA backend:

```
$ vault write cuddletech_ops/intermediate/set-signed \
> certificate=@cuddletech_ops.crt
Success! Data written to: cuddletech_ops/intermediate/set-signed
```

Awesome! Lets verify:

```
$ curl -s http://localhost:8200/v1/cuddletech_ops/ca/pem | openssl x509 -text | head -20
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            76:12:53:41:be:18:98:2c:a1:51:4a:f8:f0:bd:b4:a3:44:7e:74:59
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN=Cuddletech Root CA
        Validity
            Not Before: Jul  9 09:23:39 2016 GMT
            Not After : Jul  9 09:24:09 2017 GMT
        Subject: CN=Cuddletech Ops Intermediate CA
 ...
```

The last thing we need to do is set the CA & CRL URL’s for accessing the CA:

```
$ vault write cuddletech_ops/config/urls \
> issuing_certificates="http://10.0.0.22:8200/v1/cuddletech_ops/ca" \
> crl_distribution_points="http://10.0.0.22:8200/v1/cuddletech_ops/crl"
Success! Data written to: cuddletech_ops/config/urls
```
