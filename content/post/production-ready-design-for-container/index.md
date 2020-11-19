---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "A Production Ready Design for a Container"
subtitle: ""
summary: ""
authors: [manish-gupta]
tags: [Container, Docker]
categories: [Docker]
date: 2020-11-18T14:41:42-06:00
lastmod: 2020-11-18T14:41:42-06:00
featured: false
draft: false

# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder.
# Focal points: Smart, Center, TopLeft, Top, TopRight, Left, Right, BottomLeft, Bottom, BottomRight.
image:
  caption: ""
  focal_point: ""
  preview_only: false

# Projects (optional).
#   Associate this post with one or more of your projects.
#   Simply enter your project's folder or file name without extension.
#   E.g. `projects = ["internal-project"]` references `content/project/deep-learning/index.md`.
#   Otherwise, set `projects = []`.
projects: []
---
When we started our journey to build Keyper, we were clear from the beginning that we'd bundle keyper as a container. As we progressed with its development, we looked into various container designs. Before this, our experience with building containers was playing with a toy variety and primarily with Docker. Some containerized applications we evaluated were large (over 1GB) and had, in some cases, multiple containers. We aimed to keep the design of the Keyper container simple and its size smaller than 100MB. And, we also wanted Keyper [Twelve-Factor App](https://12factor.net/) compliant.

Although Linux was a natural choice as a base OS for the container, it took us a while to find the right Linux Distro. Most Linux distribution assumes that it is running on real hardware or a virtualized hardware and not in a container such as Docker. Some distros have a complex init system that is not suitable for a Container. We did not want to treat our Container as VM. 

When we looked around, we found many distros that could fit our bill but were bigger than the size threshold of 100MB (e.g. Ubuntu, CentOS). And, the package repository of some of the smaller distros was not that complete (e.g. many busybox based images). Researching further, we found that Alpine Linux to be a popular base OS for Docker application. The reason is its tiny base image size (5 MB) and completeness of its repository and toolset. Based on this, we decided to use Alpine as a base for Keyper.

Here is what we found when compared image sizes for various distros.

| Image  | Size  |
|--------|-------|
| Ubuntu | 72MB  |
| CentOS | 215MB |
| Debian | 114MB |
| Alpine | 5MB   |

Once we settled on the Alpine Linux, we started to think about its tooling. We did not want to use a full-fledged init system like Upstart or Systemd. While working with an OpenLDAP container, we found its service model to be pretty elegant. When we dug deeper, we found [docker-light-baseimage](https://github.com/osixia/docker-light-baseimage) as its base. We found two compelling features in this image:

1. Simple way to install services and multiple process image stacks. It uses Runit init scheme. The Runit is a cross-platform Unix init scheme with service supervision. It fits perfectly within a docker container framework keeping the size of the container small. Runit features parallelization of the start-up of system services, which can speed up the boot time of the operating system. We found light-weight service supervision a killer feature.
2. Getting environment variables from .yaml and .json files.
 
The docker-light-baseimage is based on Debian. So, we migrated its toolset to Alpine Linux.

Let us take a look at its internals.

## Image Directory Structure
This image uses four directories:

* **/container/environment**: for environment files.
* **/container/service**: for services to install, setup and run.
* **/container/service-available**: for service that may be on demand downloaded, installed, setup and run.
* **/container/tool**: for image tools.

{{% alert note %}}
As there was no need, we decided ignore ```/container/service-available```
{{% /alert %}}

This image creates another directory during the run time:

* **/container/run**: To store container run environment, state, startup files and process to run based on files in ```/container/environment``` and ```/container/service``` directories.

## Service Directory Structure

A service directory contains shell scripts, with each shell script having a specific purpose. The service directory is added in ```/container/service``` to create a service

* **my-service**: root directory
* **my-service/install.sh**: install script (not mandatory).
* **my-service/startup.sh**: startup script to setup the service when the container start (not mandatory).
* **my-service/process.sh**: process to run (not mandatory).
* **my-service/finish.sh**: finish script run when the process script exit (not mandatory).
* **my-service/...** add whatever you need!

## Tools

All container tools are located under the ```/container/tool``` directory and are linked in ```sbin/``` so they belong to the container PATH.

| Filename        | Description |
| ---------------- | ------------------- |
| run | The run tool is defined as the image ENTRYPOINT (see [Dockerfile](https://github.com/dbsentry/keyper-docker/blob/master/Dockerfile)). It set the environment and run startup scripts and images process. More information in the [Advanced User Guide](https://github.com/osixia/docker-light-baseimage#run). |
| setuser | A tool for running a command as another user. Easier to use than su, has a smaller attack vector than sudo, and unlike chpst this tool sets $HOME correctly.|
| log-helper | A simple bash tool to print message base on the log level. |
|  add-service-available | A tool to download and add services in service-available directory to the regular service directory. |
| add-multiple-process-stack | A tool to add the multiple process stack: runit, cron syslog-ng-core, and logrotate. |
| install-service | A tool that execute /container/service/install.sh and /container/service/\*/install.sh scripts. |
|  complex-bash-env | A tool to iterate through complex bash environment variables created by the run tool when a table or a list was set in environment files or environment command-line argument. |

## Keyper Container Design
Based on what we learned from the docker-light-baseimage, let us look at how we adapted this in the Keyper container design:

```console
keyper-docker
├── Dockerfile
├── LICENSE
├── Makefile
├── README.md
├── .release
├── .make-release-support
├── container
│   ├── build.sh
│   ├── environment
│   │   └── default.yaml
│   ├── service
│   │   ├── gunicorn
│   │   │   ├── install.sh
│   │   │   ├── process.sh
│   │   │   └── startup.sh
│   │   ├── nginx
│   │   │   ├── assets
│   │   │   │   ├── certs
│   │   │   │   │   ├── ca.crt
│   │   │   │   │   └── dhparam.pem
│   │   │   │   ├── etc
│   │   │   │   │   └── conf.d
│   │   │   │   │       └── default.conf
│   │   │   │   └── scripts
│   │   │   │       ├── auth.sh.txt
│   │   │   │       └── authprinc.sh.txt
│   │   │   ├── install.sh
│   │   │   ├── process.sh
│   │   │   └── startup.sh
│   │   └── slapd
│   │       ├── assets
│   │       │   ├── ldif
│   │       │   │   ├── auditlog.ldif
│   │       │   │   └── ppolicy.ldif
│   │       │   ├── schema
│   │       │   │   ├── memberof.ldif
│   │       │   │   ├── openssh-lpk.ldif
│   │       │   │   └── sudo.ldif
│   │       │   └── templates
│   │       │       ├── config.ldif.tmpl
│   │       │       └── data.ldif.tmpl
│   │       ├── install.sh
│   │       ├── process.sh
│   │       └── startup.sh
│   └── tools
│       ├── add-multiple-process-stack
│       ├── add-service-available
│       ├── complex-bash-env
│       ├── install-service
│       ├── log-helper
│       ├── run
│       ├── setuser
│       └── wait-process
├── contrib
│   └── kp1-detach.sh
└── modules
    ├── build_builder.sh
    ├── keyper
    └── keyper-fe
```

We defined three services under ```/container/service```
* **gunicorn**: To run python REST API
* **nginx**: To serve REST API and Front-End App
* **slapd**: To run OpenLDAP service

And, each service has three scripts defined:
* **install.sh**: Install script
* **process.sh**: Process to run
* **startup.sh**: startup script to set up the service when container start

We kept the ```install.sh``` script empty for all the three services for simplicity. 

Let us take a look at the startup scripts for ```gunicorn``` service.

**startup.sh**: This script sets up the environment before the service is started.
```console
#!/bin/bash -e
#############################################################################
#                       Confidentiality Information                         #
#                                                                           #
# This module is the confidential and proprietary information of            #
# DBSentry Corp.; it is not to be copied, reproduced, or transmitted in any #
# form, by any means, in whole or in part, nor is it to be used for any     #
# purpose other than that for which it is expressly provided without the    #
# written permission of DBSentry Corp.                                      #
#                                                                           #
# Copyright (c) 2020-2021 DBSentry Corp.  All Rights Reserved.              #
#                                                                           #
#############################################################################

FIRST_START_DONE="${CONTAINER_STATE_DIR}/gunicorn-first-start-done"

if [ ! -e "$FIRST_START_DONE" ]; then
  touch $FIRST_START_DONE
fi

log-helper info "Setting UID/GID for nginx to ${NGINX_UID}/${NGINX_GID}"
[ "$(id -g nginx)" -eq ${NGINX_GID} ] || groupmod -g ${NGINX_GID} nginx
[ "$(id -u nginx)" -eq ${NGINX_UID} ] || usermod -u ${NGINX_UID} -g ${NGINX_GID} nginx

cd /container/service/gunicorn/assets
[ -d keyper ] && mv keyper /var/www
cd /var/www

[ -d ${SSH_CA_DIR} ] || mkdir ${SSH_CA_DIR}

if [ "$(ls -A /container/service/gunicorn/assets/sshca | grep -v lost+found)" ]; then
  cp /container/service/gunicorn/assets/sshca/* ${SSH_CA_DIR}
fi
[ -d ${SSH_CA_DIR}/${SSH_CA_TMP_WORK_DIR} ] || mkdir ${SSH_CA_DIR}/${SSH_CA_TMP_WORK_DIR}

[ -z ${SSH_CA_KEY_TYPE} ] && SSH_CA_KEY_TYPE=rsa
log-helper info "CA KEY Type: ${SSH_CA_KEY_TYPE}."

if [ ! -e "$SSH_CA_DIR/$SSH_CA_HOST_KEY" ]; then
        log-helper info "CA Host Key does not exist. Generating one ..."
    ssh-keygen -t ${SSH_CA_KEY_TYPE} -q -N "" -f ${SSH_CA_DIR}/${SSH_CA_HOST_KEY}
fi

if [ ! -e "$SSH_CA_DIR/$SSH_CA_USER_KEY" ]; then
        log-helper info "CA User Key does not exist. Generating one ..."
    ssh-keygen -t ${SSH_CA_KEY_TYPE} -q -N "" -f ${SSH_CA_DIR}/${SSH_CA_USER_KEY}
fi

if [ ! -e "$SSH_CA_DIR/$SSH_CA_KRL_FILE" ]; then
        log-helper info "CA KRL does not exist. Generating one ..."
    ssh-keygen -k -f ${SSH_CA_DIR}/${SSH_CA_KRL_FILE}
fi

[ -d /var/log/keyper ] || mkdir /var/log/keyper
chown -R nginx:nginx /var/log/keyper /var/www/keyper ${SSH_CA_DIR}

exit 0
```

**process.sh**: This script launches the service.
```console
#!/bin/bash -e
#############################################################################
#                       Confidentiality Information                         #
#                                                                           #
# This module is the confidential and proprietary information of            #
# DBSentry Corp.; it is not to be copied, reproduced, or transmitted in any #
# form, by any means, in whole or in part, nor is it to be used for any     #
# purpose other than that for which it is expressly provided without the    #
# written permission of DBSentry Corp.                                      #
#                                                                           #
# Copyright (c) 2020-2021 DBSentry Corp.  All Rights Reserved.              #
#                                                                           #
#############################################################################

sv check /container/run/process/slapd
log-helper info "gunicorn: Starting"
exec su -s /bin/sh -c "cd /var/www/keyper; . env/bin/activate; gunicorn -w 4 'app:create_app()' --bind 127.0.0.1:8000 --user=nginx --group=nginx" nginx
log-helper info "gunicorn: Started"
```
Now, let us look at how everything comes together in a ```Dockerfile```

```console
FROM  alpine:latest AS builder
RUN   apk add --no-cache  python3                     \
                            py3-yaml                    \
                            python3-dev                 \
                            bash                        \
                            gcc                         \
                            musl-dev                    \
                            npm                         \
                            openssl                     \
                            nginx                       \
                            openldap                    \
                            openldap-dev                \
                            openldap-clients            \
                            openldap-back-mdb           \
                            openldap-overlay-memberof   \
                            openldap-overlay-ppolicy  \
                            openldap-overlay-refint   \
                            openldap-overlay-auditlog   \
                            openldap-back-monitor     
COPY  modules /container
RUN   /container/build_builder.sh

FROM  alpine:latest
RUN   apk add --no-cache  python3       \
        py3-yaml                        \
        runit                           \
        bash                            \
        shadow                          \
        openssl                         \
        nginx                           \
        openssh-keygen                  \
        openldap                        \
        openldap-clients                \
        openldap-back-mdb               \
        openldap-overlay-memberof       \
        openldap-overlay-ppolicy        \
        openldap-overlay-refint         \
        openldap-overlay-auditlog       \
        openldap-back-monitor     
COPY  container /container
COPY  --from=builder /container/out.tar.gz /container/out.tar.gz
RUN   /container/build.sh
ENTRYPOINT ["/container/tools/run"]
EXPOSE 80 443 389 636
```

Above ```Dockerfile``` creates two images. The first image is to build all the Keyper sources. The second image is the Production image that gets pushed to Docker Hub. We split into two images to keep the image size smaller (also recommended by 12-factor-app) and also because the runtime does not require many build-time packages (e.g. gcc). The entry point for the Production image is set to ```/container/tools/run```. When the container is started ```run``` sets up the environment by reading environment variables either from the command line or from the following ```default.yaml``` file and starts all the services.

```console
#############################################################################
#                       Confidentiality Information                         #
#                                                                           #
# This module is the confidential and proprietary information of            #
# DBSentry Corp.; it is not to be copied, reproduced, or transmitted in any #
# form, by any means, in whole or in part, nor is it to be used for any     #
# purpose other than that for which it is expressly provided without the    #
# written permission of DBSentry Corp.                                      #
#                                                                           #
# Copyright (c) 2020-2021 DBSentry Corp.  All Rights Reserved.              #
#                                                                           #
#############################################################################
# All environment variables used after the container first start must be 
# defined here.
#############################################################################
#
LDAP_LOG_LEVEL: 256

# Ulimit
LDAP_NOFILE: 1024

# Do not perform any chown to fix file ownership
DISABLE_CHOWN: false

# UID/GID for LDAP User
LDAP_UID: 10100 
LDAP_GID: 10100

# UID/GID for LDAP User
NGINX_UID: 10080
NGINX_GID: 10080


# Default port to bind slapd
LDAP_PORT: 389
LDAPS_PORT: 636


# Required and used for new ldap server only
LDAP_ORGANIZATION_NAME: Example Inc.
LDAP_DOMAIN: keyper.example.org

LDAP_ADMIN_PASSWORD: superdupersecret

LDAP_TLS_CA_CRT_FILENAME: ca.crt
LDAP_TLS_CRT_FILENAME: server.crt
LDAP_TLS_KEY_FILENAME: server.key
LDAP_TLS_DH_PARAM_FILENAME: dhparam.pem

LDAP_TLS_ENFORCE: false
LDAP_TLS_CIPHER_SUITE: TLSv1.2:HIGH:!aNULL:!eNULL
LDAP_TLS_PROTOCOL_MIN: 3.3
LDAP_TLS_VERIFY_CLIENT: demand

FLASK_CONFIG: prod

SSH_CA_DIR: /etc/sshca
SSH_CA_HOST_KEY: ca_host_key
SSH_CA_USER_KEY: ca_user_key
SSH_CA_KEY_TYPE: rsa
SSH_CA_KRL_FILE: ca_krl
SSH_CA_TMP_WORK_DIR: tmp
SSH_CA_TMP_DELETE_FLAG: True
```

To streamline the build process we used the following ```Makefile```:

```console
#
#   Copyright 2015  Xebia Nederland B.V.
#
#   Licensed under the Apache License, Version 2.0 (the "License");
#   you may not use this file except in compliance with the License.
#   You may obtain a copy of the License at
#
#       http://www.apache.org/licenses/LICENSE-2.0
#
#   Unless required by applicable law or agreed to in writing, software
#   distributed under the License is distributed on an "AS IS" BASIS,
#   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
#   See the License for the specific language governing permissions and
#   limitations under the License.
#
REGISTRY_HOST=docker.io
REGISTRY_HOST_QUAY=quay.io
#USERNAME=$(USER)
#NAME=$(shell basename $(CURDIR))
USERNAME=dbsentry
NAME=keyper

RELEASE_SUPPORT := $(shell dirname $(abspath $(lastword $(MAKEFILE_LIST))))/.make-release-support
IMAGE=$(REGISTRY_HOST)/$(USERNAME)/$(NAME)
IMAGE_QUAY=$(REGISTRY_HOST_QUAY)/$(USERNAME)/$(NAME)

VERSION=$(shell . $(RELEASE_SUPPORT) ; getVersion)
TAG=$(shell . $(RELEASE_SUPPORT); getTag)

SHELL=/bin/bash

DOCKER_BUILD_CONTEXT=.
DOCKER_FILE_PATH=Dockerfile
DOCKER_BUILD_ARGS=--squash

.PHONY: pre-build docker-build post-build build release patch-release minor-release major-release tag check-status check-release showver \
  push pre-push do-push post-push

build: pre-build docker-build post-build

build-quay: docker-build-quay

pre-build:


post-build:


pre-push:


post-push:

pre-push-quay:


post-push-quay:


docker-build: .release
  docker build $(DOCKER_BUILD_ARGS) -t $(IMAGE):$(VERSION) $(DOCKER_BUILD_CONTEXT) -f $(DOCKER_FILE_PATH)
  @DOCKER_MAJOR=$(shell docker -v | sed -e 's/.*version //' -e 's/,.*//' | cut -d\. -f1) ; \
  DOCKER_MINOR=$(shell docker -v | sed -e 's/.*version //' -e 's/,.*//' | cut -d\. -f2) ; \
  if [ $$DOCKER_MAJOR -eq 1 ] && [ $$DOCKER_MINOR -lt 10 ] ; then \
    echo docker tag -f $(IMAGE):$(VERSION) $(IMAGE):latest ;\
    docker tag -f $(IMAGE):$(VERSION) $(IMAGE):latest ;\
  else \
    echo docker tag $(IMAGE):$(VERSION) $(IMAGE):latest ;\
    docker tag $(IMAGE):$(VERSION) $(IMAGE):latest ; \
  fi

docker-build-quay: 
  IMAGEID=$(shell . $(RELEASE_SUPPORT) ; getImageId "$(USERNAME)/$(NAME):$(VERSION)") ;\
  docker tag $$IMAGEID $(IMAGE_QUAY):$(VERSION) ; \
  docker tag $$IMAGEID $(IMAGE_QUAY):latest ; \

.release:
  @echo "release=0.0.0" > .release
  @echo "tag=$(NAME)-0.0.0" >> .release
  @echo INFO: .release created
  @cat .release


release: check-status check-release build push

push: pre-push do-push post-push 

push-quay: pre-push-quay do-push-quay post-push-quay

do-push: 
  docker push $(IMAGE):$(VERSION)
  docker push $(IMAGE):latest

do-push-quay: 
  docker push $(IMAGE_QUAY):$(VERSION)
  docker push $(IMAGE_QUAY):latest

snapshot: build push

showver: .release
  @. $(RELEASE_SUPPORT); getVersion

tag-patch-release: VERSION := $(shell . $(RELEASE_SUPPORT); nextPatchLevel)
tag-patch-release: .release tag 

tag-minor-release: VERSION := $(shell . $(RELEASE_SUPPORT); nextMinorLevel)
tag-minor-release: .release tag 

tag-major-release: VERSION := $(shell . $(RELEASE_SUPPORT); nextMajorLevel)
tag-major-release: .release tag 

patch-release: tag-patch-release release
  @echo $(VERSION)

minor-release: tag-minor-release release
  @echo $(VERSION)

major-release: tag-major-release release
  @echo $(VERSION)


tag: TAG=$(shell . $(RELEASE_SUPPORT); getTag $(VERSION))
tag: check-status
  @. $(RELEASE_SUPPORT) ; ! tagExists $(TAG) || (echo "ERROR: tag $(TAG) for version $(VERSION) already tagged in git" >&2 && exit 1) ;
  @. $(RELEASE_SUPPORT) ; setRelease $(VERSION)
  git add .
  git commit -m "bumped to version $(VERSION)" ;
  git tag $(TAG) ;
  @ if [ -n "$(shell git remote -v)" ] ; then git push --tags ; else echo 'no remote to push tags to' ; fi

check-status:
  @. $(RELEASE_SUPPORT) ; ! hasChanges || (echo "ERROR: there are still outstanding changes" >&2 && exit 1) ;

check-release: .release
  @. $(RELEASE_SUPPORT) ; tagExists $(TAG) || (echo "ERROR: version not yet tagged in git. make [minor,major,patch]-release." >&2 && exit 1) ;
  @. $(RELEASE_SUPPORT) ; ! differsFromRelease $(TAG) || (echo "ERROR: current directory differs from tagged $(TAG). make [minor,major,patch]-release." ; exit 1)
```

With this in place, a simple ```make build``` builds the container and ```make push``` pushes the image to the Docker Hub.

{{% alert note %}}
We use the above Makefile only for the development and testing of Keyper as production build and push to Docker hub has been migrated to GitHub action.
{{% /alert %}}

## Docker Image Versioning

For the image versioning and tagging, we created a ```.release``` file with the following content. We increment the ```release``` and ```tag``` number each time a new version is released. The ```Makefile``` tags the image accordingly.

```console
release=0.2.4
tag=0.2.4
```
## Image in Action
As they say, the proof the pudding is in eating. So, let us look at why this service design works. Let us launch a Keyper image:

```console
[manish@getafix2 keyper-docker]$ docker run -p 8080:80 -p 8443:443 --env SSH_CA_KEY_TYPE=ed25519 -it dbsentry/keyper
*** CONTAINER_LOG_LEVEL = 3 (info)
*** Search service in CONTAINER_SERVICE_DIR = /container/service :
*** link /container/service/gunicorn/startup.sh to /container/run/startup/gunicorn
*** link /container/service/gunicorn/process.sh to /container/run/process/gunicorn/run
*** link /container/service/nginx/startup.sh to /container/run/startup/nginx
*** link /container/service/nginx/process.sh to /container/run/process/nginx/run
*** link /container/service/slapd/startup.sh to /container/run/startup/slapd
*** link /container/service/slapd/process.sh to /container/run/process/slapd/run
*** Set environment for startup files
*** Environment files will be proccessed in this order : 
Caution: previously defined variables will not be overriden.
/container/environment/default.yaml

To see how these files are processed and environment variables values,
run this container with '--loglevel debug'
/container/tools/run:295: YAMLLoadWarning: calling yaml.load() without Loader=... is deprecated, as the default Loader is unsafe. Please read https://msg.pyyaml.org/load for full details.
  env_vars = yaml.load(f)
*** Running /container/run/startup/gunicorn...
Setting UID/GID for nginx to 10080/10080
CA KEY Type: ed25519.
CA Host Key does not exist. Generating one ...
CA User Key does not exist. Generating one ...
CA KRL does not exist. Generating one ...
*** Running /container/run/startup/nginx...
Setting UID/GID for nginx to 10080/10080
Certificate/key does not exist. Generating a self-signed certificate ...
Generating a RSA private key
....................................................................................+++++
.............................+++++
writing new private key to '/etc/nginx/certs/server.key'
-----
*** Running /container/run/startup/slapd...
Setting UID/GID for nginx to 10080/10080
--------------------------------------------------
OpenLDAP database configuration
--------------------------------------------------
LDAP ORG: Example Inc.
LDAP DOMAIN: keyper.example.org
ADMIN PASSWD: superdupersecret
BASEDN: dc=keyper,dc=example,dc=org
--------------------------------------------------
Openldap DB and Config directories are empty...
Creating new LDAP Server
Creating OpenLDAP Database: START
_#################### 100.00% eta   none elapsed            none fast!         
Closing DB...
_#################### 100.00% eta   none elapsed            none fast!         
Closing DB...
_#################### 100.00% eta   none elapsed            none fast!         
Closing DB...
_#################### 100.00% eta   none elapsed            none fast!         
Closing DB...
_#################### 100.00% eta   none elapsed            none fast!         
Closing DB...
_#################### 100.00% eta   none elapsed            none fast!         
Closing DB...
_#################### 100.00% eta   none elapsed            none fast!         
Closing DB...
Creating OpenLDAP Database: END
*** Set environment for container process
*** Environment files will be processed in this order : 
Caution: previously defined variables will not be overridden.
/container/environment/default.yaml

To see how this files are processed and environment variables values,
run this container with '--loglevel debug'
*** Running runit daemon...
ok: run: /container/run/process/slapd: (pid 109) 0s
ok: run: /container/run/process/gunicorn: (pid 107) 0s
openldap: Starting
5fb6baa7 @(#) $OpenLDAP: slapd 2.4.50 (May  7 2020 12:49:06) $
	openldap
gunicorn: Starting
nginx: Starting
[2020-11-19 18:34:16 +0000] [115] [INFO] Starting gunicorn 20.0.4
[2020-11-19 18:34:16 +0000] [115] [INFO] Listening at: http://127.0.0.1:8000 (115)
[2020-11-19 18:34:16 +0000] [115] [INFO] Using worker: sync
[2020-11-19 18:34:16 +0000] [118] [INFO] Booting worker with pid: 118
[2020-11-19 18:34:16 +0000] [120] [INFO] Booting worker with pid: 120
[2020-11-19 18:34:16 +0000] [119] [INFO] Booting worker with pid: 119
5fb6baa8 slapd starting
[2020-11-19 18:34:16 +0000] [121] [INFO] Booting worker with pid: 121
```
List running services:
```console
/container/run/process # sv status *
run: gunicorn: (pid 107) 253s
run: nginx: (pid 108) 253s
run: slapd: (pid 109) 253s
/container/run/process # 
```
Let us see what happens when we kill the ```nginx``` process:

```console
/container/run/process # ps -ef
PID   USER     TIME  COMMAND
    1 root      0:00 {run} /usr/bin/python3 -u /container/tools/run
  103 root      0:00 /sbin/runsvdir -P /container/run/process
  104 root      0:00 runsv gunicorn
  105 root      0:00 runsv nginx
  106 root      0:00 runsv slapd
  107 root      0:00 su -s /bin/sh -c cd /var/www/keyper; . env/bin/activate; gunicorn -w 4 'app:creat
  109 nginx     0:00 /usr/sbin/slapd -h ldap:/// ldaps:/// -u nginx -g nginx -d 256
  115 nginx     0:00 {gunicorn} /var/www/keyper/env/bin/python3 /var/www/keyper/env/bin/gunicorn -w 4 
  118 nginx     0:00 {gunicorn} /var/www/keyper/env/bin/python3 /var/www/keyper/env/bin/gunicorn -w 4 
  119 nginx     0:00 {gunicorn} /var/www/keyper/env/bin/python3 /var/www/keyper/env/bin/gunicorn -w 4 
  120 nginx     0:00 {gunicorn} /var/www/keyper/env/bin/python3 /var/www/keyper/env/bin/gunicorn -w 4 
  121 nginx     0:00 {gunicorn} /var/www/keyper/env/bin/python3 /var/www/keyper/env/bin/gunicorn -w 4 
  123 root      0:00 /bin/sh
  146 root      0:00 nginx: master process /usr/sbin/nginx -g daemon off;
  149 nginx     0:00 nginx: worker process
  151 root      0:00 ps -ef
/container/run/process # pkill nginx
/container/run/process # ps -ef
PID   USER     TIME  COMMAND
    1 root      0:00 {run} /usr/bin/python3 -u /container/tools/run
  103 root      0:00 /sbin/runsvdir -P /container/run/process
  104 root      0:00 runsv gunicorn
  105 root      0:00 runsv nginx
  106 root      0:00 runsv slapd
  107 root      0:00 su -s /bin/sh -c cd /var/www/keyper; . env/bin/activate; gunicorn -w 4 'app:creat
  109 nginx     0:00 /usr/sbin/slapd -h ldap:/// ldaps:/// -u nginx -g nginx -d 256
  115 nginx     0:00 {gunicorn} /var/www/keyper/env/bin/python3 /var/www/keyper/env/bin/gunicorn -w 4 
  118 nginx     0:00 {gunicorn} /var/www/keyper/env/bin/python3 /var/www/keyper/env/bin/gunicorn -w 4 
  119 nginx     0:00 {gunicorn} /var/www/keyper/env/bin/python3 /var/www/keyper/env/bin/gunicorn -w 4 
  120 nginx     0:00 {gunicorn} /var/www/keyper/env/bin/python3 /var/www/keyper/env/bin/gunicorn -w 4 
  121 nginx     0:00 {gunicorn} /var/www/keyper/env/bin/python3 /var/www/keyper/env/bin/gunicorn -w 4 
  123 root      0:00 /bin/sh
  153 root      0:00 nginx: master process /usr/sbin/nginx -g daemon off;
  156 nginx     0:00 nginx: worker process
  157 root      0:00 ps -ef
```

If you look at the stdout of the Keyper container, you'll see the following:
```console
ok: run: /container/run/process/gunicorn: (pid 107) 491s
nginx: Starting
```
As soon as the ```nginx``` process was killed Runit restarted it.

## Summation
In this post, I explained a robust and Production-ready design for a Docker container using Alpine Linux and Runit based on the design of docker-light-baseimage. This post also demonstrates the robustness of the image created using the above methodology.

```<Shameless-Plug>```  
[Keyper](https://keyper.dbsentry.com) is an Open Source SSH Key and Certificate-Based Authentication Manager, which also acts as an SSH Certificate Authority (CA). It standardizes and centralizes the storage of SSH public keys and SSH Certificates for all Linux users in your organization. It also saves significant time and effort it takes to manage SSH keys and certificates on each Linux Server. Keyper also maintains an active Key Revocation List, which prevents the use of Key/Cert once revoked. Keyper is a lightweight container taking less than 100MB. It supports both Docker and Podman. You can be up and running within minutes instead of days.  
```</Shameless-Plug>```  

That's it, folks! Happy more secure SSH'ing.