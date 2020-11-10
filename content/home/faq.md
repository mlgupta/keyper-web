+++
# A section created with the Blank widget.
widget = "blank"  # See https://sourcethemes.com/academic/docs/page-builder/
headless = true  # This file represents a page section.
active = true  # Activate this widget? true/false
weight = 70  # Order that this section will appear.

# Note: a full width section format can be enabled by commenting out the `title` and `subtitle` with a `#`.
title = "FAQ"
subtitle = ""

[design]
  # Choose how many columns the section has. Valid values: 1 or 2.
  columns = "2"

[design.background]
  # Apply a background color, gradient, or image.
  #   Uncomment (by removing `#`) an option to apply it.
  #   Choose a light or dark text color by setting `text_color_light`.
  #   Any HTML color name or Hex value is valid.

  # Background color.
  # color = "navy"
  
  # Background gradient.
  # gradient_start = "DeepSkyBlue"
  # gradient_end = "SkyBlue"
  
  # Background image.
  # image = "image.jpg"  # Name of image in `static/media/`.
  # image_darken = 0.6  # Darken the image? Range 0-1 where 0 is transparent and 1 is opaque.

  # Text color (true=light or false=dark).
  # text_color_light = true

[design.spacing]
  # Customize the section spacing. Order is top, right, bottom, left.
  # padding = ["0px", "0px", "0px", "0px"]
  padding = ["20px", "0", "20px", "0"]


[advanced]
 # Custom CSS. 
 css_style = ""
 
 # CSS class.
 css_class = ""
+++

{{% toc no_label=true %}}

# General  
## What is Keyper?
Keyper is an SSH Key and Certificate Based Authentication Manager  
## Why Keyper?  
We, as system administrators and developers, regularly use OpenSSH's public key authentication (aka password-less login) on Linux servers or Certificate based authentication (more secure). The mechanism works based on public-key cryptography. One adds his/her RSA/DSA key to the authorized_keys file on the server. The user with the corresponding private key can login without a password. It works great until the number of servers starts to grow. It becomes a pain to manage the authorized_keys file on all the servers. Account revocation becomes a pain as well. Keyper aims to centralize all such SSH Public Keys within an organization. With Keyper, one can force key rotation, easily revoke keys, and centrally lock accounts.
## Why Certificate based authentication
SSH Key based authentication works great. Certificate based authentication eliminates few limitations associated with Key based authentication. When using certificates, both clients and servers use certificates signed by trusted third party (in this case Keyper). This takes care of long time known issue such as Trust-on-First-Use (TOFU).Instead of X.509 format, SSH uses its own simpler certificate format. 
## Why should one use Keyper for certifcate based authenticaion
There is one little inconvenience with certificates and that is users cannot sign their keys themselves as they do not have access to CA's private key. A process must be put in place to sign the keys. Once adminstrator setup a user in the Keyper system, s/he can upload keys to be signed by the CA. Keyper system generates the certificate on the fly based on restrictions (like duration of certificate, and what servers/users any user has access to).
## Is Keyper opensource?  
~~Not yet. However, we are working to get it open-sourced under GPLv2 (pending permission from our corporate overlords).~~  
Yes! We Opensourced Keyper under GPLv3 license. The source repositories are located at [keyper-docker](https://github.com/dbsentry/keyper-docker) and [keyper](https://github.com/dbsentry/keyper).  
Following components of keyper are opensourced:
* keyper REST API
* keyper docker image builder
* All artifacts related to openldap schema  

The above stack can be administored using ```curl``` CLI.  

The web based administration console for Keyper has not been opensourced. 

The above arrangement should satisfy needs of most of our users as smaller commercial customers can continue to use the web based admin console bundled with docker image upto 20 servers. After which they have option to purchase license.  
## Where can I get Keyper?  
Keyper can be downloaded from the docker [registry](https://hub.docker.com/repository/docker/dbsentry/keyper) either using docker or podman.
## I have a question/suggestion/need to report a bug, how can I contact you?  
Thanks in advance. We love suggestions/bug reports. Please drop us a line at support@dbsentry.com  
## Where is the documentation?  
All documentation is located [here](/docs/)
## Do you have a Demo/Sandbox for Keyper?
Yes we do. We have a demo system running on https://sprout.dbsentry.com. And also 4 containers running SSH that can be used for testing. Here are credentials to access the web-console:

| User ID | Password    |
| ----------| ----------- | 
| alice     | keyper      |
| bob       | keyper      |
| carol     | keyper      |
| erin      | keyper      |
| frank     | keyper      |
| grace     | keyper      |

In addition, you can test following containers running SSH using above user ids after you add your SSH public key to the web-console:

| Server                 | SSH Port    |
| -----------------------| ----------- | 
| mavrix2.dbsentry.com   | 2022        |
| mavrix3.dbsentry.com   | 3022        |
| mavrix4.dbsentry.com   | 4022        |
| mavrix5.dbsentry.com   | 5022        |

So, if you want to access server ```mavrix4.dbsentry.com``` as user ```frank```, you need to add your SSH public key (typically ```~/.ssh/id_rsa.pub or ~/.ssh/id_dsa.pub```) to user frank and then connect using ssh:
```console
$ ssh -l frank -p 4022 mavrix4.dbsentry.com
```
Any issues drop us a line at support@dbsentry.com.
{{% alert note %}}
This demo system is limited and only allows you to upload/delete SSH public keys (Keyper User Role). You do not have admin access to add/modify users/hosts/groups (Keyper Admin Role). To see admin features in action, we suggest that you download and run the docker image either using docker or podman on your Linux system.
{{% /alert %}}
{{% alert note %}}
You are sharing these demo systems with others. Please be respectful of others. We delete all uploaded keys every night.
{{% /alert %}}
## I have a question which is not answered here.
Send your question to support@dbsentry.com and we'll try to address it. You can also hang out with us on [Discord](https://discord.gg/JKpgXrYvGX)
# Technical  
## What is under the hood?  
Keyper is published as a Docker container which can also be run using podman. The stack include:
* Alpine Linux  
* OpenLDAP acting as a directory that stores all users, SSH Public Keys, and rules  
* Python Flask REST API  
* VueJS frontend app running on Nginx  
* All the above services are managed using runit  
## What SSH servers are supported by Keyper?
Any Linux server running OpenSSH 6.8 or newer should be fine. An SSH server that supports AuthorizedKeysCommand is needed.  
## What is Podman?
Per podman projetct's website: "Podman is a daemonless, open source, Linux native tool designed to make it easy to find, run, build, share and deploy applications using Open Containers Initiative (OCI) Containers and Container Images. Podman provides a command line interface (CLI) familiar to anyone who has used the Docker Container Engine. Most users can simply alias Docker to Podman (alias docker=podman) without any problems."  
Best thing we liked about podman is that one need not be root to run a container. It comes bundled with RHEL (or any RHEL based distro) by default. If you do not have it install it using yum:
```console
# yum install podman
```
## How to persist OpenLDAP data on restart?
By default, Keyper creates OpenLDAP database within container under ```/var/lib/openldap/openldap-data``` and ```/etc/openldap/slapd.d```. For data to persist after a restart, we need to present local docker volumes as a parameter. Something like this:
```console
$ docker volume create slapd.d
$ docker volume create openldap-data
$ docker run -p 80:80 -p 443:443 -p 389:389 -p 636:636 --hostname <hostname> --mount source=slapd.d,target=/etc/openldap/slapd.d --mount source=openldap-data,target=/var/lib/openldap/openldap-data -it dbsentry/keyper
```  
For more information about docker data volume, please refer to:
> https://docs.docker.com/engine/tutorials/dockervolumes/
## What is the purpose of hostname as a parameter?
Keyper uses this hostname to generate a self-signed certificate. OpenLDAP and Nginx use this certificate for secure communication. Also, this hostname gets embedded in the auth.sh script which you need to download and deploy on each Linux server.  
## I want to connect to Keyper on a different terminal to see its inside?
Great. First find the container id of the running container, and then use ```docker exec``` to connect. Something like this:
```console
$ docker ps
CONTAINER ID        IMAGE                 COMMAND                  CREATED             STATUS              PORTS                                                                                       NAMES
25b2869f1a71        dbsentry/keyper       "/container/tools/run"   21 hours ago        Up 21 hours         0.0.0.0:8080->80/tcp, 0.0.0.0:2389->389/tcp, 0.0.0.0:8443->443/tcp, 0.0.0.0:2636->636/tcp   peaceful_lewin
66d33bbdd32c        jenkinsci/blueocean   "/sbin/tini -- /usr/…"   13 days ago         Up 13 days          0.0.0.0:50000->50000/tcp, 0.0.0.0:8000->8080/tcp                                            jenkins-blueocean
$ docker exec -it 25b2869f1a71 /bin/sh
/ # ls
bin        dev        home       media      opt        root       sbin       sys        usr
container  etc        lib        mnt        proc       run        srv        tmp        var
/ # 
```
## What environment variables can I set?
Following environment variables can be set while starting the container:

| Environment Variable       | Description              | Default            |
| ---------------------------| ------------------------ | ------------------ |
| LDAP_PORT                  | ldap bind port           | 389                |
| LDAPS_PORT                 | ldaps bind port          | 636                |
| LDAP_ORGANIZATION_NAME     | Name of the Organization | Example Inc.       |
| LDAP_DOMAIN                | LDAP Domain              | keyper.example.org |
| LDAP_LDAP_ADMIN_PASSWORD   | Admin password on LDAP   | superdupersecret   |
| LDAP_TLS_CA_CRT_FILENAME   | CA Cert File Name        | ca.crt             |
| LDAP_TLS_CRT_FILENAME      | Cert File Name           | server.crt         |
| LDAP_TLS_KEY_FILENAME      | Cert Key File Name       | server.key         |
| LDAP_TLS_DH_PARAM_FILENAME | DH Param File Name       | dhparam.pem        |
| LDAP_TLS_CIPHER_SUITE      | Default Cipher Suite     | TLSv1.2:HIGH:!aNULL:!eNULL |
| FLASK_CONFIG               | Flask Config (dev/prod)  | prod               |
| HOSTNAME                   | Hostname                 | {docker generated} |
| NGINX_UID                  | linux nginx user uid     | 10080              |
| NGINX_UID                  | linux nginx user uid     | 10080              |
| SSH_CA_HOST_KEY            | CA Host Key              | ca_host_key        |
| SSH_CA_USER_KEY            | CA USER Key              | ca_user_key        |

## How can I see Debug messages for the REST API?
Running a container with ```FLASK_CONFIG=dev``` would force Flask REST API to run in debug mode.
## Where is the auditlog for OpenLDAP located
```/var/log/openldap/auditlog.ldif```. It may be a better idea to create docker volume for ```/var/log``` and mount it in the container to persist logs
## How can I backup Keyper?
As far as you have a backup for the OpenLDAP database you are good to go. For the rest, as far as you specify the same cli params things should work fine.  
## How do I use a real X.509 SSL certificate with Keyper?
The certificate is used by OpenLDAP and Nginx. You can set custom certificate at run time by mounting a directory containing those files to ```/container/service/nginx/assets/certs``` and adjust their name per the environment variables defined above.
## What is SSH Key Revocation List (KRL)?
A Key Revocation List (KRL) is a list identifying the revokes Keys and Certificates. In Keyper, a Key or Certificate when deleted gets added to the KRL. Both Key based and Certificate based SSH authentication on Keyper use KRL verification as the first step.
## How can I access the KRL File?
KRL file is located on the runing container under ```/etc/sshca/ca_krl```. It can also be downloaded using API call ```https://servername/api/krlca```


