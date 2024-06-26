---
title: SSH Certificate Authorities and Key Signing
author: Julian Patocki
tags:
    - security
    - ssh
    - keygen
    - certificates
---

## Prerequisites

* Ability to use command line tools.
* Managing content from the command line.
* Previous experience with SSH key generation helpful, but not required.
* Basic understanding of SSH and public key infrastructure helpful, but not required.
* A server running the sshd daemon.

## Introduction

The initial SSH connection with a remote host is insecure, if you cannot verify the fingerprint of the remote host's key. Using a certificate authority to sign public keys of remote hosts makes the initial connection secure for every user that trusts the CA.

CAs can also be used to sign user SSH keys. Instead of distributing the key to every remote host, one signature is sufficient to authorize user to log in to multiple servers.

## Objectives

* Improving the security of SSH connections.
* Improving the onboarding process and key management.

## Notes

* Vim is my text editor of choice, but text files can be edited using nano or any other program or method.
* Usage of `sudo` or `root` implies elevated privilages are required.

## Initial Connection

To make the initial connection secure, the prior knowledge of the key fingerprint is required. Obviously this step can be optimized and integrated in the deployment process for new hosts.

Displaying the key fingerprint on the remote host:

```
user@rocky-vm ~]$ ssh-keygen -E sha256 -l -f /etc/ssh/ssh_host_ed25519_key.pub 
256 SHA256:bXWRZCpppNWxXs8o1MyqFlmfO8aSG+nlgJrBM4j4+gE no comment (ED25519)
```

Making the initial SSH connection from the client. The key fingerprint is being displayed an can be compared to the previously aquired one:

```
[user@rocky ~]$ ssh user@rocky-vm.example.com
The authenticity of host 'rocky-vm.example (192.168.56.101)' can't be established.
ED25519 key fingerprint is SHA256:bXWRZCpppNWxXs8o1MyqFlmfO8aSG+nlgJrBM4j4+gE.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

## Creating a Certificate Authority

Creating a CA (private and public keys) and placing the public key in the `known_hosts` file of the client to identify all hosts belonging to the example.com domain:

```
[user@rocky ~]$ ssh-keygen -b 4096 -t ed25519 -f CA
[user@rocky ~]$ echo '@cert-authority *.example.com' $(cat CA.pub) >> ~/.ssh/known_hosts
```

Where:

- **-b**: key lenght in bytes
- **-t**: key type: rsa, ed25519, ecdsa...
- **-f**: output key file

Alternatively, the `known_hosts` file can be specified system wide by editing the SSH config file `/etc/ssh/ssh_config`:

```
Host *
    GlobalKnownHostsFile /etc/ssh/ssh_known_hosts
```

## Signing the public keys

Creating a user SSH key and signing it:

```
[user@rocky ~]$ ssh-keygen -b 4096 -t ed2119
[user@rocky ~]$ ssh-keygen -s CA -I user -n user -V +55w  id_ed25519.pub
```

Aquiring the server's public key over scp and signing it:

```
[user@rocky ~]$ scp user@rocky-vm.example.com:/etc/ssh/ssh_host_ed25519_key.pub .
[user@rocky ~]$ ssh-keygen -s CA -I rocky-vm -n rocky-vm.example.com -h -V +55w ssh_host_ed25519_key.pub
```

Where:

- **-s**: signing key
- **-I**: name that identifies the certificate for logging purposes
- **-n**: identifies the name (host or user, one or multiple) associated with the certificate (if not specified, certificates are valid for all users or hosts)
- **-h**: defines the certificate as a host key, as opposed to a client key
- **-V**: validity period of the certificate

## Establishing Trust

Copying the remote host's certificate for the remote host to present it along its public key while being connected to:

```
[user@rocky ~]$ scp ssh_host_ed25519_key-cert.pub root@rocky-vm.example.com:/etc/ssh/
```

Copying the CA's public key to the remote host for it to trust certificates signed by the CA:

```
[user@rocky ~]$ scp CA.pub root@rocky-vm.example.com:/etc/ssh/
```

Adding the following lines to `/etc/ssh/sshd_config` file to specify the previously copied key and certificate to use by the server and trust the CA to identify users:

```
[user@rocky ~]$ ssh user@rocky-vm.example.com
[user@rocky-vm ~]$ sudo vim /etc/ssh/sshd_config
```

```
HostKey /etc/ssh/ssh_host_ed25519_key
HostCertificate /etc/ssh/ssh_host_ed25519_key-cert.pub
TrustedUserCAKeys /etc/ssh/CA.pub
```

Restarting the sshd service on the server:

```
[user@rocky-vm ~]$ systemctl restart sshd
```

## Testing the Connection

Removing remote server's fingerprint from your `known_hosts` file and verifying the settings by establishing an SSH connection:

```
[user@rocky ~]$ ssh-keygen -R rocky-vm.example.com
[user@rocky ~]$ ssh user@rocky-vm.example.com
```

## Key Revocation

Revoking host or user keys may be crutial to the security of the whole environment. Therefore it is important to store the previously signed public keys to be able to revoke them at some point in the future.

Creating an empty revokation list and revoking the public key of user2:

```
[user@rocky ~]$ sudo ssh-keygen -k -f /etc/ssh/revokation_list.krl
[user@rocky ~]$ sudo ssh-keygen -k -f /etc/ssh/revokation_list.krl -u /path/to/user2_id_ed25519.pub
```

Copying the revokation list to the remote host and specifying it in the `sshd_config` file:

```
[user@rocky ~]$ scp /etc/ssh/revokation_list.krl root@rocky-vm.example.com:/etc/ssh/
[user@rocky ~]$ ssh user@rocky-vm.example.com
[user@rocky ~]$ sudo vim /etc/ssh/sshd_config
```

The following line specifies the revokation list:

```
RevokedKeys /etc/ssh/revokation_list.krl
```

Restarting the SSHD daemon is required for the configuration to reload:

```
[user@rocky-vm ~]$ sudo systemctl restart sshd
```

User2 gets rejected by the server:

```
[user2@rocky ~]$ ssh user2@rocky-vm.example.com
user2@rocky-vm.example.com: Permission denied (publickey,gssapi-keyex,gssapi-with-mic).
```

Server keys can also be revoked:

```
[user@rocky ~]$ sudo ssh-keygen -k -f /etc/ssh/revokation_list.krl -u /path/to/ssh_host_ed25519_key.pub
```

The following lines in `/etc/ssh/ssh_config` applies the host revokation list system wide:

```
Host *
        RevokedHostKeys /etc/ssh/revokation_list.krl
```

Trying to connect to the host results in the following:

```
[user@rocky ~]$ ssh user@rocky-vm.example.com
Host key ED25519-CERT SHA256:bXWRZCpppNWxXs8o1MyqFlmfO8aSG+nlgJrBM4j4+gE revoked by file /etc/ssh/revokation_list.krl
```

Maintaining and updating the revokation lists is important and may be automated to assure the most recent revokation lists are accessible by all hosts and users.

## Conclusion

SSH is one of the most useful protocols to manage remote servers. Implemeting certificate authorities may be very helpful, especially in larger environments with multiple servers and users.
It is also important to maintain revokation lists. It ensures that compromised keys may be revoked easily without replacing the whole key infrastructure.