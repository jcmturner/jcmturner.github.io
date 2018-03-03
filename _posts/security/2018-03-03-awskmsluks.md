---
layout: post
title:  "AWS KMS LUKS Service - Key management for on-premise volume encryption"
date:   2018-03-03 12:00:00 +0000
categories: security
tags: aws kms luks
published: true
---

## Use Case
The AWS KMS LUKS service is designed to secure on-premise storage block devices from the following risks:
* Loss of a volume's physical medium
* Data access by the storage service administrators
* Physical access to the storage service's infrastructure being used as a means to access the data

It is not intended for AWS KMS LUKS to address the risk of access to the data via a privileged user on the host Linux operating to which the volume is mounted.

LUKS (Linux Unified Key Setup) is the standard for disk encryption on Linux. 
The challenge with LUKS is the management and security of passphrases used as keys to encrypt the volume.
AWS KMS LUKS aims to solve this issue by using AWS' KMS to generate and manage keys used as LUKS passphrases.

For details on AWS' KMS see: https://aws.amazon.com/kms/
A knowledge of KMS is required to understand the AWS KMS LUKS service design.

## Architecture
The AWS KMS LUKS has a server side component that proxies and secures the interaction with AWS services (KMS and S3).
The server also implements security checks on requests to access data keys used to open encrypted volumes.
A REST API is exposed for creating new data keys and decrypting data keys.
When data keys are created a backup copy is stored in an S3 bucket to ensure durability of keys.
If keys are lost from the AWS KMS LUKS clients they can be restored from this bucket.

The client to the server is a single binary that runs on the host which has the volume attached.
This is used to initialise the encryption of new volumes and decrypt and open volumes.
When initialising LUKS volume encryption the client will request a new data key from the server and use this to set up the encryption.
When openning a volume the client will take the locally stored encrypted data key and request the server to decrypt it.
The decrypted data key will then be used to open the volume. Systemd is used to call this client on boot to open all encrypted volumes on the host.

## Key Structure
AWS KMS LUKS uses KMS to generate data keys which are used as the passphrases to LUKS volume encryption.

AWS KMS LUKS uses a key structure which includes the following:
* The ARN of the AWS KMS master key used to generate and encrypt the data key.
* The data key used as the passphrase for LUKS.
  * Plaintext - removed before written to disk or the S3 bucket.
  * Encrypted - written to disk and S3.
* The encryption context used with KMS to encrypt the data key.
* A list of host names which are authorized to decrypt the encrypted form of the data key.
* A digital signature to ensure the integrity of the key structure.

## Security
The security principle behind AWS KMS LUKS is that multiple targets would need to be simultaneously compromised into order to access the data on the volume using a host other than the one intended.

The AWS KMS LUKS server has the following security features to prevent an encrypted volume being compromised:
* Plaintext keys are never written to disk or S3.
* Communication with the server uses TLS.
* Key decryption requests must pass the following checks:
  * Authentication via Kerberos.
  * Authorization against a trusted user list (optional)
  * Authorization via group membership (available when Microsoft Active Directory is used as the KDC) (optional)
  * The IP address of the requester must match a DNS lookup of one of the host names in an authorized host list in the key structure.
  * Each key structure is digitally signed using the KMS master key and validated to prevent tampering with the authorized host list.
  
In order to compromise a volume by decrypting it on another host the following would be required:
* Possession of the encrypted volume - requires privileged access to the hosting infrastructure.
* Possession of the encrypted key - requires root access on the original host.
* Possession of the Kerberos credential - requires root access on the original host or compromise of the KDC.
* Compromise of the authorized host list to permission the host being used to access the volume's data. This would require one of:
  * Compromise of the DNS service to set an existing authorized host name to resolve to the IP address of the host being used to compromise the volume.
  * Access to the KMS master key to be able to re-sign an updated key structure.