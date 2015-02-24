# GO - Key Management Service

A REST based Key Management Service written in GO.

[![GoDoc](https://godoc.org/github.com/Inflatablewoman/go-kms?status.svg)](https://godoc.org/github.com/Inflatablewoman/go-kms)
[![Build Status](https://travis-ci.org/Inflatablewoman/go-kms.svg)](https://travis-ci.org/Inflatablewoman/go-kms)
[![Coverage Status](https://coveralls.io/repos/Inflatablewoman/go-kms/badge.svg?branch=master)](https://coveralls.io/r/Inflatablewoman/go-kms?branch=master)

## What is GO-KMS?

GO-KMS is my attempt at creating a Key Management Service in GO.  Modelled extensively on AWS KMS behaviour, the API is used for symmetrical key management.  

GO-KMS authentication is done using [HMAC-SHA256](http://en.wikipedia.org/wiki/Hash-based_message_authentication_code) over HTTPS.  

The default crypto provider is based on [AES](http://en.wikipedia.org/wiki/Advanced_Encryption_Standard) and a key size of 256bits using the [GCM cipher](http://en.wikipedia.org/wiki/Galois/Counter_Mode) to provide confidentiality as well as authentication.  Keys are encrypted and stored on disk, using a master key which is derived using [PBKDF2](http://en.wikipedia.org/wiki/PBKDF2) from a passphrase.

### GO-KMS - Command Line Interface

GO-KMS-CLI is a command line interface which can be used to manage and interact with go-kms.  The project can be found here: [https://github.com/Inflatablewoman/go-kms-cli](https://github.com/Inflatablewoman/go-kms-cli)

## Features

- AES Key store
- Keys encrypted while at rest
- Shared-key Authentication
- Playground for HSM support via SoftHSM2

## Todo

- Some documentation about the keys
- RSA Encryption provider
- Full HSM provider support 

## How-To

To run get the project...

```
go get github.com/Inflatablewoman/go-kms
```

You need to set the following variables:

```
export GOKMS_AUTH_KEY=/path/to/auth.key
export GOKMS_CRYPTO_PROVIDER=softhsm | gokms
export GOKMS_HOST=localhost
export GOKMS_PORT=8020
export GOKMS_SSL_CERT=/path/to/ssl_cert.pem
export GOKMS_SSL_KEY=/path/to/ssl_key.pem
```

If the Crypto_Provider is set to gokms then you also need to set:

```
export GOKMS_KSMC_PATH=/path/to/keys/
export GOKMS_KSMC_PASSPHRASE="a very long Passphrase that will be used for key derivation"
```

## Authorization

Authorization is done via a *Authorization* header sent in a request.  Anonymous requests are not allowed.  To authenticate a request, you must sign the request with the shared key when making the request and pass that signature as part of the request.  

Here you can see an example of a Authorization header
```
Authorization=RvPtP0QB7iIun1ehwheD4YUo7+fYfw7/ywl+HsC5Ddk=
```

You construct the signature is built in the following format:

```
authRequestSig = method + "\n" +
                 Date + "\n" +
                 resource
```

This would result in the following signature to be signed:

```
POST\nWed, 28 Jan 2015 10:42:13 UTC\n/api/v1/go-ksm
```

Note that you MUST past the same date value in the request.  Date should be supplied in UTC using RFC1123 format.

```
x-kms-date=Wed, 28 Jan 2015 10:42:13 UTC
```

  The signature must be exactly in the same order and include the new line character.  

Now encode the signature using the [HMAC-SHA256](http://en.wikipedia.org/wiki/Hash-based_message_authentication_code) algorithm using the shared key.

This will result in a key like this:
```
RvPtP0QB7iIun1ehwheD4YUo7+fYfw7/ywl+HsC5Ddk="
```

Example go code to create the signature

```go
date := time.Now().UTC().Format(time.RFC1123) // UTC time
request.Header.Add("x-kms-date", date)

authRequestKey := fmt.Sprintf("%s\n%s\n%s", method, date, resource)

// See package http://golang.org/pkg/crypto/hmac/ on how golang creates hmacs
hmac := crypto.GetHmac256(authRequestKey, SharedKey)  

request.Header.Add("Authorization", hmac)
```

## Good resources:

### KMS

- AWS KMS: https://d0.awsstatic.com/whitepapers/KMS-Cryptographic-Details.pdf
- MS Key Vault: https://msdn.microsoft.com/en-US/library/azure/dn903623

### PKCS#11

- https://github.com/miekg/pkcs11
- http://www-01.ibm.com/support/knowledgecenter/linuxonibm/com.ibm.linux.z.lxce/lxce_linklib_object_samples.html

### HMS

- https://www.opendnssec.org/
- SofthHsm2 only supports RSA encryption: https://wiki.opendnssec.org/display/SoftHSM/v2+Requirements
- https://wiki.opendnssec.org/display/SoftHSMDOCS/SoftHSM+Documentation+v2.0
- http://docs.oracle.com/javase/7/docs/technotes/guides/security/p11guide.html

## Install notes if building SofthHsm2 support:

Add some missing things needed to build, pkcs11

```
sudo yum install libtool-ltdl-devel
```

Install softhsm2
```
sudo yum-config-manager --add-repo http://copr-fe.cloud.fedoraproject.org/coprs/pspacek/softhsm2/
sudo yum install softhsm2
```

Create a place to store the tokens

```
mkdir /home/keithball/Documents/tokens
```

Create/change config file to reflect new token location

```
# SoftHSM v2 configuration file

directories.tokendir = /home/keithball/Documents/tokens
objectstore.backend = file

# ERROR, WARNING, INFO, DEBUG
log.level = DEBUG
```

Create initial token...

```
export SOFTHSM2_CONF=$PWD/softhsm2.conf
softhsm2-util --init-token --slot 0 --label "My token 1"
```

Check it worked ok

```
softhsm2-util --show-slots
```
