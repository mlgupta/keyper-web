---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Openldap Backends and Overlays"
subtitle: ""
summary: ""
authors: [manish-gupta]
tags: [OpenLDAP]
categories: [OpenLDAP]
date: 2020-09-18T13:36:09-05:00
lastmod: 2020-09-18T13:36:09-05:00
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
If you are planning to use OpenLDAP or any LDAP/Directory for that matter for your project. You should think hard and come up with the structure and features you are going to use before installation. It is a lot easier to configure things in the beginning than later.

Backends and overlays in OpenLDAP are one such configuration. 

## Backends
As per OpenLDAP's manual, "Backends do the actual work of storing or retrieving data in response to LDAP requests. Backends may be compiled statically into slapd, or when module support is enabled, they may be dynamically loaded."

If we compare with RDBMS, OpenLDAP backends are what table storage technology is to RDBMS. You should spend enough time in the beginning (before installation) and think through what backends you are going to need and configure them early on. Having said that requirements change, and you may need to add a backend. We are going to cover here one such backend configuration.

### Configure Monitor Backend
Monitor backend once enables is automatically generated and dynamically maintained by slapd with information about the running status of the daemon.

As stated earlier, the backend can be either compiled statically with OpenLDAP or loaded as a module. We are going to use a module-based approach.

To load the monitor module, create an LDIF file like this and name it monitor.ldif:

```console
dn: cn=module{0},cn=config
objectClass: olcModuleList
changetype: modify
add: olcModuleLoad
olcModuleLoad: {0}back_monitor.so
```

Import the ldif file:

```console
$ ldapmodify -x -D "cn=config" -W -f ./monitor.ldif
```

Create another LDIF file apply_monitor.ldif:

```console
dn: olcdatabase=monitor,cn=config
objectclass: olcDatabaseConfig
olcDatabase: monitor
olcAccess: to dn.subtree=cn=monitor by users read
```

Apply the monitor backend by import above file:

```console
$ ldapadd -x -D "cn=config" -W -f ./apply_monitor.ldif
```

## Overlays
Overlays are components that sits on top of Backends and provide hooks to various functions. In the RDBMS parly, overlay would be views with stored procedures.

### Configure memberof overlay
The memberof attribute dynamically maintaines list of group of an entry. It updates an attribute dynamically whenever membership attributes change for an entry.

To load memberof overlay, create a LDIF file like this and name it memberof.ldif:

```console
dn: cn=module{0},cn=config
objectClass: olcModuleList
changetype: modify
add: olcModuleLoad
olcModuleLoad: {0}memberof.so
```

Import the ldif file:

```console
$ ldapmodify -x -D "cn=config" -W -f ./memberof.ldif
```

Create another LDIF file apply_memberof.ldif:

```console
dn: olcOverlay=memberof,olcDatabase={1}mdb,cn=config
objectClass: olcConfig
objectClass: olcMemberOf
objectClass: olcOverlayConfig
objectClass: top
olcOverlay: memberof
olcMemberOfRefint: TRUE
olcMemberOfDangling: ignore
olcMemberOfGroupOC: groupOfNames
olcMemberOfMemberAD: member
olcMemberOfMemberOfAD: memberOf
```

Apply the monitor backend by import above file:

```console
$ ldapadd -x -D "cn=config" -W -f ./apply_memberof.ldif
```

After configuration of memberof overlay, if you add a member attribute to a group (groupOfNames) by adding a full DN, a corresponding memberof attribute would get created an entry that was added to the group.