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
Keyper is an SSH Key Manager  
## Why Keyper?  
We as system administrators and developes regularly use OpenSSH's public key authentication (aka password-less login) on linux servers. The mechanism works based on public key cryptography. One adds his/her RSA/DSA key to the authorized_keys file on the server. The user with the corresponding private key can login without password. This works great. However, when the numbe of servers start to grow it becomes pain to manage authorized_keys file on all the servers. Account revocation becomes a pain as well. Keyper aims to centralize all such SSH Public Keys wihin an organization. With keyper one can force key rotation, easily revoke keys, centrally lock accounts.
## Is Keyper opensource?  
Not yet, however we are working to get it opensourced under GPLv2 (pending permission from our corporate overlords).  
## Where can I get Keyper?  
Keyper can be downloaded from the docker [registry](https://hub.docker.com/repository/docker/dbsentry/keyper) either using docker or podman.
## I have a question/suggestion/need to report a bug how can I contact you?  
Thanks in advance. We love suggestions/bug reports. Please drop us a line at support@dbsentry.com  
## Where is the documentation?  
All documentation is located [here](/docs/)
## I have a question, which is not answered here.
Send your question to support@dbsentry.com and we'll try to address it.
# Technical  
## What is under the hood?  
Keyper is published as docker container, which can also be run using podman. The stack include:
* Alpine Linux
* OpenLDAP acting as directory that stores all users, SSH Public Keys, and rules
* Python Flask REST API
* VueJS frontend app running on nginx.  
* All the above services are managed using runit.  
## What SSH servers are supported by keyper?
Any linux server running OpenSSH 6.8 or newer should be fine. Basically, a SSH server with AuthorizedKeysCommand is needed.  
## How to persist openldap data on restart?
By default, keyper creates openldap database within container under /var/lib/openldap/openldap-data and /etc/openldap/slapd.d. For data to persis after restart, we need to present local docker volumes as parameter. Something like this:
```console
$ docker volume create slapd.d
$ docker volume create openldap-data
$ docker run -p 80:80 -p 443:443 -p 389:389 -p 636:636 --hostname <hostname> --mount source=slapd.d,target=/etc/openldap/slapd.d --mount source=openldap-data,target=/var/lib/openldap/openldap-data -it dbsentry/keyper
```  
For more information about docker data volume, please refer to:
> https://docs.docker.com/engine/tutorials/dockervolumes/
## What is the purpose of hostname as parameter?
Keyper uses this hostname to generate self-signed certificate. In addition, this hostname gets embedded in the auth.sh script which you need to download and deploy on each linux server.  
## I want to connect to keyper on different terminal to see its inside?
Great. First fine the container id of the running container, and then use "docker exec" to connect. Something like this:
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
## How can I see Debug messages for the REST API?
Runing container with FLASK_CONFIG=dev would force Flask REST API to run in debug mode.
## Where are auditlog for openldap located
/var/log/openldap/auditlog.ldif
It may be a better idea to create docker volume for /var/log and mount it in container to persist logs
## How can I backup Keyper?
As far as you have backup for the openldap database you are good to go. For the rest, as far as you specify the same cli params things should work fine.
## How do I use real SSL certificate with Keyper?
Certificate is used by slapd and nginx. You can set custom certificate at run time, by mounting a directory containing those files to /container/service/nginx/assets/certs and adjust their name per the environment variables defined above.

