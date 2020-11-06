---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Anatomy of OpenSSH Key Revocation List (KRL) File"
subtitle: ""
summary: ""
authors: [manish-gupta]
tags: []
categories: [SSH]
date: 2020-11-04T21:17:10-06:00
lastmod: 2020-11-04T21:17:10-06:00
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
## Introduction
When I started working on the SSH Certificate feature for Keyper, the question of certificate revocation and its verification came up. I searched but could not find a lightweight python library for this (aim is to keep the docker image smaller than 100MB). As a first iteration, I implemented this using Python subprocess and ```ssh-keygen```. ```ssh-keygen``` is an awesome utility and considered a Swiss Army Knife for SSH Keys. The same utility helps you when you are either setting up a certificate authority (CA) or OpenSSH Key Revocation List (KRL) file.

## KRL Basics
A KRL can be created using ```ssh-keygen```:
```console
[manish@getafix sshca]$ ssh-keygen -k -f ca_krl

[manish@getafix sshca]$ ls -l
total 4
-rw-r--r-- 1 manish manish 44 Nov  5 11:09 ca_krl
[manish@getafix sshca]$
```

A Key or a Certificate is revoked by adding them to the KRL using ```ssh-keygen```:
```console
[manish@getafix sshca]$ ls -l
total 12
-rw-r--r-- 1 manish manish   44 Nov  5 11:09 ca_krl
-rw-r--r-- 1 manish manish 2009 Nov  5 11:13 id_rsa-cert.pub
-rw-r--r-- 1 manish manish  568 Nov  5 11:13 id_rsa.pub

[manish@getafix sshca]$ ssh-keygen -k -u -f ca_krl id_rsa.pub
Revoking from id_rsa.pub

[manish@getafix sshca]$ ssh-keygen -k -u -f ca_krl id_rsa-cert.pub
Revoking from id_rsa-cert.pub
[manish@getafix sshca]$ 
```

A Key or a Certificate revocation can be checked using ```ssh-keygen```. The output is as follows when the Key/Certificate is not in the KRL:
```console
[manish@getafix sshca]$ ssh-keygen -Q -f ca_krl id_rsa.pub
id_rsa.pub (manish@getafix): ok

[manish@getafix sshca]$ ssh-keygen -Q -f ca_krl id_rsa-cert.pub
id_rsa-cert.pub (manish@getafix): ok
[manish@getafix sshca]$ 
```

And, as follows when Key/Certificate is in the KRL:
```console
[manish@getafix sshca]$ ssh-keygen -Q -f ca_krl id_rsa.pub
id_rsa.pub (manish@getafix): REVOKED

[manish@getafix sshca]$ ssh-keygen -Q -f ca_krl id_rsa-cert.pub
id_rsa-cert.pub (manish@getafix): REVOKED
[manish@getafix sshca]$ 
```

## ssh-keygen limitation
Although ```ssh-keygen``` can revoke keys or certificates using their fingerprint or serial number, it needs full Key or the Certificate for the KRL verification. As a result, when using Keyper, each SSH server needs to be configured to send Public Key or the Certificate (configured using ```%k``` and ```%t``` in the ```sshd_config``` file). There is always an option to periodically copy the KRL file to each SSH server so that it performs local KRL lookup. I was not happy with sending the full Key or the Certificate as part of the API call during each authentication. But could neither find a way to perform KRL lookup using fingerprint or serial number using ```ssh-keygen``` nor find a lightweight Python library for it. So, I decided to write a KRL lookup myself in python. This post is about what I learned while doing this.

```man ssh-keygen``` defines OpenSSH format Key Revocation Lists (KRLs) as "binary files specify keys or certificates to be revoked using a compact format, taking as little as one bit per certificate if they are being revoked by serial number." 

## KRL Anatomy
To understand its internal structure I started with the OpenSSH source code. File ```krl.c``` has the following relvant definition:

```c
/*
 * Trees of revoked serial numbers, key IDs and keys. This allows
 * quick searching, querying and producing lists in canonical order.
 */

/* Tree of serial numbers. XXX make smarter: really need a real sparse bitmap */
struct revoked_serial {
        u_int64_t lo, hi;
        RB_ENTRY(revoked_serial) tree_entry;
};
static int serial_cmp(struct revoked_serial *a, struct revoked_serial *b);
RB_HEAD(revoked_serial_tree, revoked_serial);
RB_GENERATE_STATIC(revoked_serial_tree, revoked_serial, tree_entry, serial_cmp);

/* Tree of key IDs */
struct revoked_key_id {
        char *key_id;
        RB_ENTRY(revoked_key_id) tree_entry;
};
static int key_id_cmp(struct revoked_key_id *a, struct revoked_key_id *b);
RB_HEAD(revoked_key_id_tree, revoked_key_id);
RB_GENERATE_STATIC(revoked_key_id_tree, revoked_key_id, tree_entry, key_id_cmp);

/* Tree of blobs (used for keys and fingerprints) */
struct revoked_blob {
        u_char *blob;
        size_t len;
        RB_ENTRY(revoked_blob) tree_entry;
};
static int blob_cmp(struct revoked_blob *a, struct revoked_blob *b);
RB_HEAD(revoked_blob_tree, revoked_blob);
RB_GENERATE_STATIC(revoked_blob_tree, revoked_blob, tree_entry, blob_cmp);

/* Tracks revoked certs for a single CA */
struct revoked_certs {
        struct sshkey *ca_key;
        struct revoked_serial_tree revoked_serials;
        struct revoked_key_id_tree revoked_key_ids;
        TAILQ_ENTRY(revoked_certs) entry;
};
TAILQ_HEAD(revoked_certs_list, revoked_certs);

struct ssh_krl {
        u_int64_t krl_version;
        u_int64_t generated_date;
        u_int64_t flags;
        char *comment;
        struct revoked_blob_tree revoked_keys;
        struct revoked_blob_tree revoked_sha1s;
        struct revoked_blob_tree revoked_sha256s;
        struct revoked_certs_list revoked_certs;
};
```
I started with ```struct ssh_krl``` and after spending a couple of hours trying to read and understand the OpenSSH code, my eyes were glazing. So, I went back back to the internet search to see if anyone has already figured this out. I found this [page](https://github.com/openssh/openssh-portable/blob/master/PROTOCOL.krl).

## KRL File Format
```c
This describes the key/certificate revocation list format for OpenSSH.

1. Overall format

The KRL consists of a header and zero or more sections. The header is:

#define KRL_MAGIC		0x5353484b524c0a00ULL  /* "SSHKRL\n\0" */
#define KRL_FORMAT_VERSION	1

	uint64	KRL_MAGIC
	uint32	KRL_FORMAT_VERSION
	uint64	krl_version
	uint64	generated_date
	uint64	flags
	string	reserved
	string	comment

Where "krl_version" is a version number that increases each time the KRL
is modified, "generated_date" is the time in seconds since 1970-01-01
00:00:00 UTC that the KRL was generated, "comment" is an optional comment
and "reserved" an extension field whose contents are currently ignored.
No "flags" are currently defined.

Following the header are zero or more sections, each consisting of:

	byte	section_type
	string	section_data

Where "section_type" indicates the type of the "section_data". An exception
to this is the KRL_SECTION_SIGNATURE section, that has a slightly different
format (see below).

The available section types are:

#define KRL_SECTION_CERTIFICATES		1
#define KRL_SECTION_EXPLICIT_KEY		2
#define KRL_SECTION_FINGERPRINT_SHA1		3
#define KRL_SECTION_SIGNATURE			4
#define KRL_SECTION_FINGERPRINT_SHA256		5

2. Certificate section

These sections use type KRL_SECTION_CERTIFICATES to revoke certificates by
serial number or key ID. The consist of the CA key that issued the
certificates to be revoked and a reserved field whose contents is currently
ignored.

	string ca_key
	string reserved

Where "ca_key" is the standard SSH wire serialisation of the CA's
public key. Alternately, "ca_key" may be an empty string to indicate
the certificate section applies to all CAs (this is most useful when
revoking key IDs).

Followed by one or more sections:

	byte	cert_section_type
	string	cert_section_data

The certificate section types are:

#define KRL_SECTION_CERT_SERIAL_LIST	0x20
#define KRL_SECTION_CERT_SERIAL_RANGE	0x21
#define KRL_SECTION_CERT_SERIAL_BITMAP	0x22
#define KRL_SECTION_CERT_KEY_ID		0x23

2.1 Certificate serial list section

This section is identified as KRL_SECTION_CERT_SERIAL_LIST. It revokes
certificates by listing their serial numbers. The cert_section_data in this
case contains:

	uint64	revoked_cert_serial
	uint64	...

This section may appear multiple times.

2.2. Certificate serial range section

These sections use type KRL_SECTION_CERT_SERIAL_RANGE and hold
a range of serial numbers of certificates:

	uint64	serial_min
	uint64	serial_max

All certificates in the range serial_min <= serial <= serial_max are
revoked.

This section may appear multiple times.

2.3. Certificate serial bitmap section

Bitmap sections use type KRL_SECTION_CERT_SERIAL_BITMAP and revoke keys
by listing their serial number in a bitmap.

	uint64	serial_offset
	mpint	revoked_keys_bitmap

A bit set at index N in the bitmap corresponds to revocation of a keys with
serial number (serial_offset + N).

This section may appear multiple times.

2.4. Revoked key ID sections

KRL_SECTION_CERT_KEY_ID sections revoke particular certificate "key
ID" strings. This may be useful in revoking all certificates
associated with a particular identity, e.g. a host or a user.

	string	key_id[0]
	...

This section must contain at least one "key_id". This section may appear
multiple times.

3. Explicit key sections

These sections, identified as KRL_SECTION_EXPLICIT_KEY, revoke keys
(not certificates). They are less space efficient than serial numbers,
but are able to revoke plain keys.

	string	public_key_blob[0]
	....

This section must contain at least one "public_key_blob". The blob
must be a raw key (i.e. not a certificate).

This section may appear multiple times.

4. SHA1/SHA256 fingerprint sections

These sections, identified as KRL_SECTION_FINGERPRINT_SHA1 and
KRL_SECTION_FINGERPRINT_SHA256, revoke plain keys (i.e. not
certificates) by listing their hashes:

	string	public_key_hash[0]
	....

This section must contain at least one "public_key_hash". The hash blob
is obtained by taking the SHA1 or SHA256 hash of the public key blob.
Hashes in this section must appear in numeric order, treating each hash
as a big-endian integer.

This section may appear multiple times.
...
```

## KRL Internals in action

The above clarified a lot. However, I still wasn't clear about how would a parser figure the length of any string in the KRL file? (for e.g. ```string section_data```) I decided to start looking into the KRL file itself. I started with a freshly generated KRL file.

```console
[manish@getafix sshca]$ ssh-keygen -k -f ca_krl

[manish@getafix sshca]$ hexdump -C ca_krl
00000000  53 53 48 4b 52 4c 0a 00  00 00 00 01 00 00 00 00  |SSHKRL..........|
00000010  00 00 00 00 00 00 00 00  5f a4 36 97 00 00 00 00  |........_.6.....|
00000020  00 00 00 00 00 00 00 00  00 00 00 00              |............|
0000002c
[manish@getafix sshca]$ 
```

It was pretty straightforward to parse header values:

KRL_MAGIC:
```console
00000000  {{< hl >}} 53 53 48 4b 52 4c 0a 00 {{< /hl >}}  00 00 00 01 00 00 00 00  |SSHKRL..........|
00000010  00 00 00 00 00 00 00 00  5f a4 36 97 00 00 00 00  |........_.6.....|
00000020  00 00 00 00 00 00 00 00  00 00 00 00              |............|
```

KRL_FORMAT_VERSION:
```console
00000000  53 53 48 4b 52 4c 0a 00  {{< hl >}} 00 00 00 01 {{< /hl >}} 00 00 00 00  |SSHKRL..........|
00000010  00 00 00 00 00 00 00 00  5f a4 36 97 00 00 00 00  |........_.6.....|
00000020  00 00 00 00 00 00 00 00  00 00 00 00              |............|
```

krl_version:
```console
00000000  53 53 48 4b 52 4c 0a 00  00 00 00 01 {{< hl >}} 00 00 00 00 {{< /hl >}}   |SSHKRL..........|
00000010  {{< hl >}}00 00 00 00{{< /hl >}} 00 00 00 00  5f a4 36 97 00 00 00 00  |........_.6.....|
00000020  00 00 00 00 00 00 00 00  00 00 00 00              |............|
```

generated_date:
```console
00000000  53 53 48 4b 52 4c 0a 00  00 00 00 01 00 00 00 00  |SSHKRL..........|
00000010  00 00 00 00 {{< hl >}} 00 00 00 00  5f a4 36 97{{< /hl >}}  00 00 00 00  |........_.6.....|
00000020  00 00 00 00 00 00 00 00  00 00 00 00              |............|
```

flags:
```console
00000000  53 53 48 4b 52 4c 0a 00  00 00 00 01 00 00 00 00  |SSHKRL..........|
00000010  00 00 00 00 00 00 00 00  5f a4 36 97 {{< hl >}} 00 00 00 00 {{< /hl >}}  |........_.6.....|
00000020  {{< hl >}}00 00 00 00 {{< /hl >}} 00 00 00 00  00 00 00 00              |............|
```

But ```00 00 00 00  00 00 00 00``` for ```reserved``` and ```comment``` confused me.

I added a key to the KRL hoping that may clarify things further. And, clarify it did!

Notice ```02``` right after ```reserved``` and ```comment```:

```console
[manish@getafix sshca]$ ssh-keygen -k -u -f ca_krl id_rsa.pub
Revoking from id_rsa.pub

[manish@getafix sshca]$ hexdump -C ca_krl 
00000000  53 53 48 4b 52 4c 0a 00  00 00 00 01 00 00 00 00  |SSHKRL..........|
00000010  00 00 00 00 00 00 00 00  5f a4 36 97 00 00 00 00  |........_.6.....|
00000020  00 00 00 00 00 00 00 00  00 00 00 00 {{<hl>}}02{{</hl>}} 00 00 01  |................|
00000030  9b 00 00 01 97 00 00 00  07 73 73 68 2d 72 73 61  |.........ssh-rsa|
00000040  00 00 00 03 01 00 01 00  00 01 81 00 c0 bb 7c b3  |..............|.|
00000050  6f 37 88 c6 dc e5 dc df  1b 81 59 a1 9b 79 9d 4e  |o7........Y..y.N|
00000060  63 b7 d0 9d 7f f2 19 e4  23 7d 9c 36 9a 08 01 39  |c.......#}.6...9|
00000070  83 bf 93 a6 b5 e8 cb 78  21 89 49 e9 9b 52 15 a4  |.......x!.I..R..|
00000080  03 5a 4a cd 26 b3 ed 7d  46 7e 8d 1e 74 35 64 43  |.ZJ.&..}F~..t5dC|
00000090  23 70 0b 7c b3 a7 f6 6c  b8 0b ca 27 5f fd 56 de  |#p.|...l...'_.V.|
000000a0  7d 1e 53 90 49 b6 fc d6  0c b5 cc 71 2d 8b b7 27  |}.S.I......q-..'|
000000b0  4d 49 e0 f4 e6 1f ea 6a  02 e0 32 23 d2 75 56 12  |MI.....j..2#.uV.|
000000c0  04 2b 22 3a 27 12 f7 ca  14 a5 9e 25 93 a2 b2 93  |.+":'......%....|
000000d0  61 f5 dc 70 6d d7 ee f3  0c a0 cb 70 f8 ab d9 71  |a..pm......p...q|
000000e0  56 7e 8d 10 7c 2b ff 40  10 9f 3e 0f 60 41 92 90  |V~..|+.@..>.`A..|
000000f0  82 c7 5b c8 5c ad de 56  25 11 c9 7e a2 aa 54 7b  |..[.\..V%..~..T{|
00000100  1f ef f2 42 0f 80 11 cb  83 b4 ec 0c 33 9f 8f bd  |...B........3...|
00000110  bb 15 31 8c 6a 59 15 4a  9d 6e 5d 5c 02 1a bc 0d  |..1.jY.J.n]\....|
00000120  fb b1 bf 0e 09 e9 65 28  57 ad b8 87 a1 64 7f 62  |......e(W....d.b|
00000130  2d e1 e4 dc e8 c8 97 f8  d1 9b 9b 25 92 03 cb 7b  |-..........%...{|
00000140  84 85 27 aa 87 d8 21 9d  98 58 23 e6 ad 45 07 3b  |..'...!..X#..E.;|
00000150  8b 3a 22 cf f9 db d6 7d  d5 57 c2 f7 f8 bf 62 56  |.:"....}.W....bV|
00000160  27 ce 8b 6d 1b a5 46 50  53 e1 c8 6e fc 24 06 f1  |'..m..FPS..n.$..|
00000170  db e3 0c 2a 45 ce 2e 4b  87 b3 e7 c4 77 04 6f f2  |...*E..K....w.o.|
00000180  91 89 42 45 34 df cc b0  af a8 bc 16 c5 7c c2 42  |..BE4........|.B|
00000190  1b 6c d3 52 3a 40 2b 01  9e 24 31 5e 28 4d 82 60  |.l.R:@+..$1^(M.`|
000001a0  ff 81 d7 37 fe 09 08 6a  46 d1 5b 1d e8 c9 39 58  |...7...jF.[...9X|
000001b0  d5 01 9e 4b a2 0c db 6f  f9 c5 d3 56 2f 49 7a c0  |...K...o...V/Iz.|
000001c0  fc 6e 6c d8 ce 96 28 bf  e5 6d 5d 43              |.nl...(..m]C|
000001cc
```

```02``` means it is the beginning of the ```KRL_SECTION_EXPLICIT_KEY``` section. The next 8 bytes clarifies things further:

```console
00000000  53 53 48 4b 52 4c 0a 00  00 00 00 01 00 00 00 00  |SSHKRL..........|
00000010  00 00 00 00 00 00 00 00  5f a4 36 97 00 00 00 00  |........_.6.....|
00000020  00 00 00 00 00 00 00 00  00 00 00 00 02 {{<hl>}}00 00 01{{</hl>}}  |................|
00000030  {{<hl>}}9b 00 00 01 97{{</hl>}} 00 00 00  07 73 73 68 2d 72 73 61  |.........ssh-rsa|
00000040  00 00 00 03 01 00 01 00  00 01 81 00 c0 bb 7c b3  |..............|.|
00000050  6f 37 88 c6 dc e5 dc df  1b 81 59 a1 9b 79 9d 4e  |o7........Y..y.N|
00000060  63 b7 d0 9d 7f f2 19 e4  23 7d 9c 36 9a 08 01 39  |c.......#}.6...9|
00000070  83 bf 93 a6 b5 e8 cb 78  21 89 49 e9 9b 52 15 a4  |.......x!.I..R..|
00000080  03 5a 4a cd 26 b3 ed 7d  46 7e 8d 1e 74 35 64 43  |.ZJ.&..}F~..t5dC|
00000090  23 70 0b 7c b3 a7 f6 6c  b8 0b ca 27 5f fd 56 de  |#p.|...l...'_.V.|
000000a0  7d 1e 53 90 49 b6 fc d6  0c b5 cc 71 2d 8b b7 27  |}.S.I......q-..'|
000000b0  4d 49 e0 f4 e6 1f ea 6a  02 e0 32 23 d2 75 56 12  |MI.....j..2#.uV.|
000000c0  04 2b 22 3a 27 12 f7 ca  14 a5 9e 25 93 a2 b2 93  |.+":'......%....|
000000d0  61 f5 dc 70 6d d7 ee f3  0c a0 cb 70 f8 ab d9 71  |a..pm......p...q|
000000e0  56 7e 8d 10 7c 2b ff 40  10 9f 3e 0f 60 41 92 90  |V~..|+.@..>.`A..|
000000f0  82 c7 5b c8 5c ad de 56  25 11 c9 7e a2 aa 54 7b  |..[.\..V%..~..T{|
00000100  1f ef f2 42 0f 80 11 cb  83 b4 ec 0c 33 9f 8f bd  |...B........3...|
00000110  bb 15 31 8c 6a 59 15 4a  9d 6e 5d 5c 02 1a bc 0d  |..1.jY.J.n]\....|
00000120  fb b1 bf 0e 09 e9 65 28  57 ad b8 87 a1 64 7f 62  |......e(W....d.b|
00000130  2d e1 e4 dc e8 c8 97 f8  d1 9b 9b 25 92 03 cb 7b  |-..........%...{|
00000140  84 85 27 aa 87 d8 21 9d  98 58 23 e6 ad 45 07 3b  |..'...!..X#..E.;|
00000150  8b 3a 22 cf f9 db d6 7d  d5 57 c2 f7 f8 bf 62 56  |.:"....}.W....bV|
00000160  27 ce 8b 6d 1b a5 46 50  53 e1 c8 6e fc 24 06 f1  |'..m..FPS..n.$..|
00000170  db e3 0c 2a 45 ce 2e 4b  87 b3 e7 c4 77 04 6f f2  |...*E..K....w.o.|
00000180  91 89 42 45 34 df cc b0  af a8 bc 16 c5 7c c2 42  |..BE4........|.B|
00000190  1b 6c d3 52 3a 40 2b 01  9e 24 31 5e 28 4d 82 60  |.l.R:@+..$1^(M.`|
000001a0  ff 81 d7 37 fe 09 08 6a  46 d1 5b 1d e8 c9 39 58  |...7...jF.[...9X|
000001b0  d5 01 9e 4b a2 0c db 6f  f9 c5 d3 56 2f 49 7a c0  |...K...o...V/Iz.|
000001c0  fc 6e 6c d8 ce 96 28 bf  e5 6d 5d 43              |.nl...(..m]C|
000001cc
```

Notice that ```00 00 01 97``` (407) is 4 less then ```00 00 01 9b``` (411). It also seem that the Key is stored base64 decoded. To verify that lets us base64 decode the Key and compare:

```console
[manish@getafix sshca]$ echo "AAAAB3NzaC1yc2EAAAADAQABAAABgQDAu3yzbzeIxtzl3N8bgVmhm3mdTmO30J1/8hnkI32cNpoIATmDv5OmtejLeCGJSembUhWkA1pKzSaz7X1Gfo0edDVkQyNwC3yzp/ZsuAvKJ1/9Vt59HlOQSbb81gy1zHEti7cnTUng9OYf6moC4DIj0nVWEgQrIjonEvfKFKWeJZOispNh9dxwbdfu8wygy3D4q9lxVn6NEHwr/0AQnz4PYEGSkILHW8hcrd5WJRHJfqKqVHsf7/JCD4ARy4O07Awzn4+9uxUxjGpZFUqdbl1cAhq8Dfuxvw4J6WUoV624h6Fkf2It4eTc6MiX+NGbmyWSA8t7hIUnqofYIZ2YWCPmrUUHO4s6Is/529Z91VfC9/i/YlYnzottG6VGUFPhyG78JAbx2+MMKkXOLkuHs+fEdwRv8pGJQkU038ywr6i8FsV8wkIbbNNSOkArAZ4kMV4oTYJg/4HXN/4JCGpG0Vsd6Mk5WNUBnkuiDNtv+cXTVi9JesD8bmzYzpYov+VtXUM=" | base64 -d | hexdump -C
00000000  00 00 00 07 73 73 68 2d  72 73 61 00 00 00 03 01  |....ssh-rsa.....|
00000010  00 01 00 00 01 81 00 c0  bb 7c b3 6f 37 88 c6 dc  |.........|.o7...|
00000020  e5 dc df 1b 81 59 a1 9b  79 9d 4e 63 b7 d0 9d 7f  |.....Y..y.Nc....|
00000030  f2 19 e4 23 7d 9c 36 9a  08 01 39 83 bf 93 a6 b5  |...#}.6...9.....|
00000040  e8 cb 78 21 89 49 e9 9b  52 15 a4 03 5a 4a cd 26  |..x!.I..R...ZJ.&|
00000050  b3 ed 7d 46 7e 8d 1e 74  35 64 43 23 70 0b 7c b3  |..}F~..t5dC#p.|.|
00000060  a7 f6 6c b8 0b ca 27 5f  fd 56 de 7d 1e 53 90 49  |..l...'_.V.}.S.I|
00000070  b6 fc d6 0c b5 cc 71 2d  8b b7 27 4d 49 e0 f4 e6  |......q-..'MI...|
00000080  1f ea 6a 02 e0 32 23 d2  75 56 12 04 2b 22 3a 27  |..j..2#.uV..+":'|
00000090  12 f7 ca 14 a5 9e 25 93  a2 b2 93 61 f5 dc 70 6d  |......%....a..pm|
000000a0  d7 ee f3 0c a0 cb 70 f8  ab d9 71 56 7e 8d 10 7c  |......p...qV~..||
000000b0  2b ff 40 10 9f 3e 0f 60  41 92 90 82 c7 5b c8 5c  |+.@..>.`A....[.\|
000000c0  ad de 56 25 11 c9 7e a2  aa 54 7b 1f ef f2 42 0f  |..V%..~..T{...B.|
000000d0  80 11 cb 83 b4 ec 0c 33  9f 8f bd bb 15 31 8c 6a  |.......3.....1.j|
000000e0  59 15 4a 9d 6e 5d 5c 02  1a bc 0d fb b1 bf 0e 09  |Y.J.n]\.........|
000000f0  e9 65 28 57 ad b8 87 a1  64 7f 62 2d e1 e4 dc e8  |.e(W....d.b-....|
00000100  c8 97 f8 d1 9b 9b 25 92  03 cb 7b 84 85 27 aa 87  |......%...{..'..|
00000110  d8 21 9d 98 58 23 e6 ad  45 07 3b 8b 3a 22 cf f9  |.!..X#..E.;.:"..|
00000120  db d6 7d d5 57 c2 f7 f8  bf 62 56 27 ce 8b 6d 1b  |..}.W....bV'..m.|
00000130  a5 46 50 53 e1 c8 6e fc  24 06 f1 db e3 0c 2a 45  |.FPS..n.$.....*E|
00000140  ce 2e 4b 87 b3 e7 c4 77  04 6f f2 91 89 42 45 34  |..K....w.o...BE4|
00000150  df cc b0 af a8 bc 16 c5  7c c2 42 1b 6c d3 52 3a  |........|.B.l.R:|
00000160  40 2b 01 9e 24 31 5e 28  4d 82 60 ff 81 d7 37 fe  |@+..$1^(M.`...7.|
00000170  09 08 6a 46 d1 5b 1d e8  c9 39 58 d5 01 9e 4b a2  |..jF.[...9X...K.|
00000180  0c db 6f f9 c5 d3 56 2f  49 7a c0 fc 6e 6c d8 ce  |..o...V/Iz..nl..|
00000190  96 28 bf e5 6d 5d 43                              |.(..m]C|
00000197
```

Bingo! That is a match. Moreover, the decoded Key is of length ```00000197``` (407). So, ```00 00 01 9b``` (411) means the length of the section. And, ```00 00 01 97``` (407) the length of the Key. So, what happen if we add another Key?

```console
[manish@getafix sshca]$ ssh-keygen -k -u -f ca_krl id_ed25519.pub
Revoking from id_ed25519.pub

[manish@getafix sshca]$ hexdump -C ca_krl 
00000000  53 53 48 4b 52 4c 0a 00  00 00 00 01 00 00 00 00  |SSHKRL..........|
00000010  00 00 00 00 00 00 00 00  5f a4 36 97 00 00 00 00  |........_.6.....|
00000020  00 00 00 00 00 00 00 00  00 00 00 00 02 {{<hl>}}00 00 01{{</hl>}}  |................|
00000030  {{<hl>}}d2 00 00 01 97{{</hl>}} 00 00 00  07 73 73 68 2d 72 73 61  |.........ssh-rsa|
00000040  00 00 00 03 01 00 01 00  00 01 81 00 c0 bb 7c b3  |..............|.|
00000050  6f 37 88 c6 dc e5 dc df  1b 81 59 a1 9b 79 9d 4e  |o7........Y..y.N|
00000060  63 b7 d0 9d 7f f2 19 e4  23 7d 9c 36 9a 08 01 39  |c.......#}.6...9|
00000070  83 bf 93 a6 b5 e8 cb 78  21 89 49 e9 9b 52 15 a4  |.......x!.I..R..|
00000080  03 5a 4a cd 26 b3 ed 7d  46 7e 8d 1e 74 35 64 43  |.ZJ.&..}F~..t5dC|
00000090  23 70 0b 7c b3 a7 f6 6c  b8 0b ca 27 5f fd 56 de  |#p.|...l...'_.V.|
000000a0  7d 1e 53 90 49 b6 fc d6  0c b5 cc 71 2d 8b b7 27  |}.S.I......q-..'|
000000b0  4d 49 e0 f4 e6 1f ea 6a  02 e0 32 23 d2 75 56 12  |MI.....j..2#.uV.|
000000c0  04 2b 22 3a 27 12 f7 ca  14 a5 9e 25 93 a2 b2 93  |.+":'......%....|
000000d0  61 f5 dc 70 6d d7 ee f3  0c a0 cb 70 f8 ab d9 71  |a..pm......p...q|
000000e0  56 7e 8d 10 7c 2b ff 40  10 9f 3e 0f 60 41 92 90  |V~..|+.@..>.`A..|
000000f0  82 c7 5b c8 5c ad de 56  25 11 c9 7e a2 aa 54 7b  |..[.\..V%..~..T{|
00000100  1f ef f2 42 0f 80 11 cb  83 b4 ec 0c 33 9f 8f bd  |...B........3...|
00000110  bb 15 31 8c 6a 59 15 4a  9d 6e 5d 5c 02 1a bc 0d  |..1.jY.J.n]\....|
00000120  fb b1 bf 0e 09 e9 65 28  57 ad b8 87 a1 64 7f 62  |......e(W....d.b|
00000130  2d e1 e4 dc e8 c8 97 f8  d1 9b 9b 25 92 03 cb 7b  |-..........%...{|
00000140  84 85 27 aa 87 d8 21 9d  98 58 23 e6 ad 45 07 3b  |..'...!..X#..E.;|
00000150  8b 3a 22 cf f9 db d6 7d  d5 57 c2 f7 f8 bf 62 56  |.:"....}.W....bV|
00000160  27 ce 8b 6d 1b a5 46 50  53 e1 c8 6e fc 24 06 f1  |'..m..FPS..n.$..|
00000170  db e3 0c 2a 45 ce 2e 4b  87 b3 e7 c4 77 04 6f f2  |...*E..K....w.o.|
00000180  91 89 42 45 34 df cc b0  af a8 bc 16 c5 7c c2 42  |..BE4........|.B|
00000190  1b 6c d3 52 3a 40 2b 01  9e 24 31 5e 28 4d 82 60  |.l.R:@+..$1^(M.`|
000001a0  ff 81 d7 37 fe 09 08 6a  46 d1 5b 1d e8 c9 39 58  |...7...jF.[...9X|
000001b0  d5 01 9e 4b a2 0c db 6f  f9 c5 d3 56 2f 49 7a c0  |...K...o...V/Iz.|
000001c0  fc 6e 6c d8 ce 96 28 bf  e5 6d 5d 43 {{<hl>}}00 00 00 33{{</hl>}}  |.nl...(..m]C...3|
000001d0  00 00 00 0b 73 73 68 2d  65 64 32 35 35 31 39 00  |....ssh-ed25519.|
000001e0  00 00 20 49 ff e8 a8 54  79 59 6b cc 73 e1 dc 05  |.. I...TyYk.s...|
000001f0  7a 47 07 4a 0e 84 56 11  f2 39 8f 50 e9 f1 11 27  |zG.J..V..9.P...'|
00000200  b2 65 63                                          |.ec|
00000203
```

The section length increased from ```0000019b``` (411) to ```000001d2``` (466) and the 4 bytes right before the newly added Key is ```00 00 00 33`` (51), which refers to the length of the new Key.

```console
[manish@getafix sshca]$ echo "AAAAC3NzaC1lZDI1NTE5AAAAIEn/6KhUeVlrzHPh3AV6RwdKDoRWEfI5j1Dp8REnsmVj" | base64 -d | hexdump -C
00000000  00 00 00 0b 73 73 68 2d  65 64 32 35 35 31 39 00  |....ssh-ed25519.|
00000010  00 00 20 49 ff e8 a8 54  79 59 6b cc 73 e1 dc 05  |.. I...TyYk.s...|
00000020  7a 47 07 4a 0e 84 56 11  f2 39 8f 50 e9 f1 11 27  |zG.J..V..9.P...'|
00000030  b2 65 63                                          |.ec|
00000033
```

So, it seems that the 4 bytes right before any string is its length. To confirm this hypothesis,  Let us add a Certificate to an empty KRL.

```console
[manish@getafix sshca]$ ssh-keygen -k -f ca_krl

[manish@getafix sshca]$ ssh-keygen -k -u -f ca_krl id_rsa-cert.pub
Revoking from id_rsa-cert.pub

[manish@getafix sshca]$ hexdump -C ca_krl 
00000000  53 53 48 4b 52 4c 0a 00  00 00 00 01 00 00 00 00  |SSHKRL..........|
00000010  00 00 00 00 00 00 00 00  5f a4 4e b8 00 00 00 00  |........_.N.....|
00000020  00 00 00 00 00 00 00 00  00 00 00 00 {{<hl>}}01 00 00 01{{</hl>}}  |................|
00000030  {{<hl>}}ac 00 00 01 97{{</hl>}} 00 00 00  07 73 73 68 2d 72 73 61  |.........ssh-rsa|
00000040  00 00 00 03 01 00 01 00  00 01 81 00 c0 a2 f4 3e  |...............>|
00000050  a2 a0 ba 1f 1b 31 6e 67  e6 dd dc f4 e6 7f ba 52  |.....1ng.......R|
00000060  b8 8a 89 3f 4e 3d d3 2e  cb 56 c4 83 bf 7c 5c 97  |...?N=...V...|\.|
00000070  7e ac a1 67 0a ee 2a 43  d9 e4 16 53 10 2a da 59  |~..g..*C...S.*.Y|
00000080  1f 4e ef a6 38 88 ba 1b  41 0d 08 21 43 83 b1 ce  |.N..8...A..!C...|
00000090  c0 13 65 5f 59 78 28 e6  1d 35 4d ba 58 00 64 39  |..e_Yx(..5M.X.d9|
000000a0  aa 32 a6 89 56 6c ba 63  2a 76 2a b4 78 17 36 b4  |.2..Vl.c*v*.x.6.|
000000b0  00 c6 b2 83 5b 2f 2a 0b  de 9a 28 34 59 bf 85 b5  |....[/*...(4Y...|
000000c0  ea 33 e7 73 24 4f e8 f7  c5 4d a3 c1 5a b7 9a 95  |.3.s$O...M..Z...|
000000d0  38 13 e4 c6 7b 47 c4 0c  f0 9d 0c aa be 1e 1b 60  |8...{G.........`|
000000e0  e1 6d ef 52 97 6e 6d d7  17 76 f6 dc 40 39 1a 13  |.m.R.nm..v..@9..|
000000f0  3a e9 d8 e4 0e e7 86 8c  b2 72 c7 f6 db 1e cc a8  |:........r......|
00000100  6f 70 a0 ca f7 01 b7 9b  6a a1 88 56 75 4e a3 9d  |op......j..VuN..|
00000110  dd c2 4c 60 4a 56 61 bd  85 da d2 48 57 96 cd b6  |..L`JVa....HW...|
00000120  a2 fe 46 72 61 c5 b4 15  69 88 0f 9b fd cb 8c 80  |..Fra...i.......|
00000130  3d bf fe 8d f1 f9 5e 10  4c 7a 18 84 66 e6 5f e8  |=.....^.Lz..f._.|
00000140  3f 2c fc fd a4 24 04 63  64 f8 8d 53 fe bd 4b 10  |?,...$.cd..S..K.|
00000150  5b 78 0b 10 27 de a2 77  33 60 24 a9 fe 04 fe 79  |[x..'..w3`$....y|
00000160  19 cd d3 cd 24 9d 92 8b  0a 92 9f 4b 73 77 1e 8e  |....$......Ksw..|
00000170  9d 56 d8 3c 37 a6 23 f9  58 52 9c bb 8f f6 f7 f8  |.V.<7.#.XR......|
00000180  ea 29 90 c1 a2 82 f1 df  f3 42 0e 00 9a ef 01 ad  |.).......B......|
00000190  cf 0c be 5c 2f 67 01 76  a6 44 19 9e 54 36 c9 1f  |...\/g.v.D..T6..|
000001a0  6a a0 32 dc 45 db e1 9d  f9 ba 01 3a 00 9f 0f 41  |j.2.E......:...A|
000001b0  73 37 06 4f 52 4f 04 b8  d1 86 46 54 57 54 9c 63  |s7.ORO....FTWT.c|
000001c0  cf 98 70 ee 95 d0 81 d0  ab 0e f2 ef 00 00 00 00  |..p.............|
000001d0  {{<hl>}}20 00 00 00 08{{</hl>}} 00 00 00  00 00 00 04 d2           | ............|
000001dd
```

Notice ```01``` right after ```comment```,  which translates to ```KRL_SECTION_CERTIFICATES```. Right after that is ```000001ac``` (428) corresponding to the length of the section. This is then followed by ```00000197``` (407). Now, that seems familiar, as this was the length when we added an RSA Key to the KRL. It is actually the CA Public Key used to sign the certificate. A reserved string (```00000000```) follows the CA Key. The next is a byte, showing the certificate section type. In this case, it is ```20``` , which means ```KRL_SECTION_CERT_SERIAL_LIST```. Per the documentation, the list of ```uint64``` (8 bytes each) should follow. The 4 bytes right after indicates the length of the list (i.e. ```00000008```). And, 8 bytes after that indicate the serial number of the certificate (i.e 1234)

```console
[manish@getafix sshca]$ ssh-keygen -L -f id_rsa-cert.pub 
id_rsa-cert.pub:
        Type: ssh-rsa-cert-v01@openssh.com user certificate
        Public key: RSA-CERT SHA256:RaHZyPzIZ0MfR0o7RkhFp4H7u67ByfL2WLVxx9OtdFY
        Signing CA: RSA SHA256:K1vwispwIJgFLOgsetpEXiiOUztYYClYATIB27qUvuI (using ssh-rsa)
        Key ID: "manish"
        Serial: 1234
        Valid: from 2020-11-05T13:11:00 to 2021-11-04T14:12:37
        Principals: 
                manish
        Critical Options: (none)
        Extensions: 
                permit-X11-forwarding
                permit-agent-forwarding
                permit-port-forwarding
                permit-pty
                permit-user-rc
```

Armed with the above knowledge, I set out to write a python code to parse the KRL file. Please note that in Keyper we use fingerprint to revoke a Key and Serial No. to revoke a Certificate. So, the code only parses the relevant sections and ignores the rest of the file. Also, it is not fully optimized yet. If you have a need to parse other sections, you can use the above knowledge and extend the code further.

When ```SSHKRL``` is initialized, it parses a KRL and creates a Python dictionary containing the list of revoked Key Fingerprints and revoked Certificate serial numbers. It has two methods ```is_key_revoked(key_hash)``` and ```is_cert_revoked(cert_serial)```. ```key_hash``` and ```cert_serial``` are Key fingerprint and certificate serial number respectively. Both methods return either ```True``` or ```False``` based on whether or not a given Key/Certificate has been revoked or not.

With this class in place, now SSH servers need not send a full Key or Certificate to perform KRL lookup.

## KRL Parser

```python
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
import struct
import base64
from flask import current_app as app
from ..resources.errors import KeyperError, errors

"""
Implements SSH KRL Lookup.
"""

class SSHKRL(object):
    ''' SSHKRL Class '''
    ca_krl_file = ''
    krl_buf_len = 0
    krl = {}

    def __init__(self):
        app.logger.debug("Enter")
        self.ca_dir = app.config["SSH_CA_DIR"]
        self.ca_krl_file = self.ca_dir + "/" + app.config["SSH_CA_KRL_FILE"]

        try:
            with open(self.ca_krl_file, mode="rb") as krl_file:
                krlbuf = krl_file.read()
                self.krl_buf_len = len(krlbuf)
                krl_buf_ptr = 0

                app.logger.debug("KRL File Size: " + str(self.krl_buf_len))

                # Parse headers
                if (self.krl_buf_len < krl_buf_ptr + 44):
                    raise KeyperError(errors["KRLParseError"].get("msg"), errors["KRLParseError"].get("status"))

                self.krl["krl_sig"] = krlbuf[krl_buf_ptr:8]
                krl_buf_ptr += 8

                self.krl["krl_format_version"] = krlbuf[krl_buf_ptr:krl_buf_ptr+4]
                krl_buf_ptr += 4

                self.krl["krl_version"] = krlbuf[krl_buf_ptr:krl_buf_ptr+8]
                krl_buf_ptr += 8
                self.krl["krl_date"] = krlbuf[krl_buf_ptr:krl_buf_ptr+8]
                krl_buf_ptr += 8
                self.krl["krl_flags"] = krlbuf[krl_buf_ptr:krl_buf_ptr+8]
                krl_buf_ptr += 8

                krl_buf_ptr, reserved_string = self.read_string_from_buf(krlbuf, krl_buf_ptr)
                krl_buf_ptr, self.krl["krl_comment"] = self.read_string_from_buf(krlbuf, krl_buf_ptr)

                # Parse sections
                while (krl_buf_ptr < self.krl_buf_len):
                    section_type = struct.unpack('c', krlbuf[krl_buf_ptr:krl_buf_ptr+1])[0]
                    krl_buf_ptr += 1

                    krl_buf_ptr, section_data = self.read_string_from_buf(krlbuf, krl_buf_ptr)

                    if (section_type == b'\x01'):
                        section_ptr = 0
                        section_data_len = len(section_data)

                        if ("krl_certs" not in self.krl):
                            self.krl["krl_certs"] = []
                        while (section_ptr < section_data_len):
                            krl_certs = {}
                            section_ptr, krl_certs["ca_key"] = self.read_string_from_buf(section_data, section_ptr)
                            section_ptr, reserved_string = self.read_string_from_buf(section_data, section_ptr)

                            cert_section_type = struct.unpack('c', section_data[section_ptr:section_ptr+1])[0]
                            section_ptr += 1

                            section_ptr, cert_serial_list = self.read_string_from_buf(section_data, section_ptr)

                            if (cert_section_type == b'\x20'):
                                cert_serial_list_ptr = 0
                                cert_serial_list_len = len(cert_serial_list)

                                krl_certs["cert_serial_list"] = []
                                while (cert_serial_list_ptr < cert_serial_list_len):
                                    krl_certs["cert_serial_list"].append(struct.unpack('>q', cert_serial_list[cert_serial_list_ptr:cert_serial_list_ptr+8])[0])
                                    app.logger.debug("Cert Serial No: " + str(struct.unpack('>q', cert_serial_list[cert_serial_list_ptr:cert_serial_list_ptr+8])[0]))
                                    cert_serial_list_ptr += 8
                                app.logger.debug("Cert Serial List Size: " + str(len(krl_certs["cert_serial_list"])))

                            self.krl["krl_certs"].append(krl_certs)
                    elif (section_type == b'\x02'):
                        section_ptr = 0
                        section_data_len = len(section_data)
                        if ("krl_keys" not in self.krl):
                            self.krl["krl_keys"] = []
                        
                        while (section_ptr < section_data_len):
                            section_ptr, krl_key = self.read_string_from_buf(section_data, section_ptr)
                            self.krl["krl_keys"].append(krl_key)

                        app.logger.debug("KRL Keys Size: " + str(len(self.krl["krl_keys"])))
                    elif (section_type == b'\x05'):
                        section_ptr = 0
                        section_data_len = len(section_data)
                        if ("krl_key_hash" not in self.krl):
                            self.krl["krl_key_hash"] = []
                        while (section_ptr < section_data_len):
                            section_ptr, krl_key_hash = self.read_string_from_buf(section_data, section_ptr)
                            self.krl["krl_key_hash"].append(krl_key_hash)
                            app.logger.debug("Key Hash: " + str(krl_key_hash))

                        app.logger.debug("KRL Key Hash Size: " + str(len(self.krl["krl_key_hash"])))
        except OSError as e:
            app.logger.error("OS error: " + str(e))
            raise KeyperError(errors["OSError"].get("msg"), errors["OSError"].get("status"))

        app.logger.debug("Exit")

    def read_string_from_buf(self, buf, ptr):
        ''' Returns a string from buffer '''
        app.logger.debug("Enter")

        result_string = None
        result_ptr = ptr
        buf_len = len(buf)

        string_size = struct.unpack('>i', buf[result_ptr:result_ptr+4])[0]
        result_ptr += 4

        if (buf_len < result_ptr + string_size):
            app.logger.error("KRL Parse Error at section. PTR: " + str(result_ptr))
            raise KeyperError(errors["KRLParseError"].get("msg"), errors["KRLParseError"].get("status"))

        result_string = buf[result_ptr:result_ptr+string_size]
        result_ptr += string_size

        app.logger.debug("Exit")
        return result_ptr, result_string

    def is_key_revoked(self, key_hash):
        ''' Checks if key hash in KRL '''
        app.logger.debug("Enter")

        rc = False

        try:
            app.logger.debug("key_hash: " + key_hash)
            key_hash_split = key_hash.split(":")[1]
            app.logger.debug("key_hash split: " + key_hash_split)

            missing_padding = len(key_hash_split) + 4 - (len(key_hash_split) % 4) 
            app.logger.debug("missing padding: " + str(missing_padding))
            key_hash_split = key_hash_split.ljust(missing_padding, '=')
            app.logger.debug("key_hash_split:" + key_hash_split)
            
            decoded_hash = base64.b64decode(key_hash_split)
            if ("krl_key_hash" in self.krl):
                if (decoded_hash in self.krl["krl_key_hash"]):
                    rc = True
        except OSError as e:
            app.logger.error("OS error: " + str(e))
            raise KeyperError(errors["OSError"].get("msg"), errors["OSError"].get("status"))

        app.logger.debug("Exit")
        return rc

    def is_cert_revoked(self, cert_serial):
        ''' Checks if cert serial in KRL '''
        app.logger.debug("Enter")

        rc = False

        try:
            app.logger.debug("cert_serial: " + str(cert_serial))
            if ("krl_certs" in self.krl):
                for krl_cert in self.krl["krl_certs"]:
                    app.logger.debug("Revoked serial list: " + str(krl_cert["cert_serial_list"]))
                    if (cert_serial in krl_cert["cert_serial_list"]):
                        rc = True
                        break
        except OSError as e:
            app.logger.error("OS error: " + str(e))
            raise KeyperError(errors["OSError"].get("msg"), errors["OSError"].get("status"))

        app.logger.debug("Exit")
        return rc
```

## Summation
In this post, I explained the internal structure for OpenSSH Key Revocation List (KRL) format with examples. This post also presents a Python class for KRL file parsing and performs KRL lookups without having to use ```ssh-keygen```.

```<Shameless-Plug>```  
Although the use of certificates results in more secure SSH authentication, SSH CA adds the burden of ssh certificate management. To ease that burden one can use a centralized system such as  [Keyper](https://keyper.dbsentry.com). Keyper is an Open Source SSH Key and Certificate-Based Authentication Manager. Keyper acts as an SSH Certificate Authority (CA) and it standardizes and centralizes the storage of SSH public keys and SSH Certificates for all Linux users in your organization saving significant time and effort it takes to manage SSH public keys and certificates on each Linux Server. Keyper also maintains an active Key Revocation List, which prevents the use of Key/Cert once revoked. Keyper is a lightweight container taking less than 100MB. It is launched either using Docker or Podman. You can be up and running within minutes instead of days.  
```</Shameless-Plug>```  

Thats it folks! Happy more secure SSH'ing.