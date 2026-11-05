---
date: 2026-11-03
title: "WIP - Fulcrum | HackTheBox - WIP"
categories: ["Web Pentesting", "Active Directory Pentesting"]
tags: ["HackTheBox", "LFI", "SSRF", "API", "File Upload", "PowerShell"]
author: "Corey Farley"
summary: ""
showToc: true
cover:
  image: cover.png
---

`Fulcrum` is a machine that presents an unusually wide attack surface up front: six open ports spanning multiple nginx-fronted web apps, a phpMyAdmin instance, and a JSON API advertising itself as `Fulcrum-API Beta`. The initial foothold centers on the application listening on port 4, which exposes an `index.php?page=` parameter and a file upload endpoint.


## Enumeration

Full port nmap scan first:

```
┌──(corey㉿kali)-[~/htb/Fulcrum]
└─$ nmap 10.129.136.254 -sS -p- -T4 
Starting Nmap 7.99 ( https://nmap.org ) at 2026-11-03 05:03 -0500
Nmap scan report for 10.129.136.254
Host is up (0.046s latency).
Not shown: 65529 closed tcp ports (reset)
PORT      STATE SERVICE
4/tcp     open  unknown
22/tcp    open  ssh
80/tcp    open  http
88/tcp    open  kerberos-sec
9999/tcp  open  abyss
56423/tcp open  unknown
```

Now aggressive on the present services:

```
┌──(corey㉿kali)-[~/htb/Fulcrum]
└─$ nmap 10.129.136.254 -A -p4,22,80,88,9999,56423 -T4
Starting Nmap 7.99 ( https://nmap.org ) at 2026-11-03 05:05 -0500
Nmap scan report for 10.129.136.254
Host is up (0.051s latency).

PORT      STATE SERVICE VERSION
4/tcp     open  http    nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: Site doesnt have a title (text/html; charset=UTF-8).
22/tcp    open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 48:ad:d5:b8:3a:9f:bc:be:f7:e8:20:1e:f6:bf:de:ae (RSA)
|   256 b7:89:6c:0b:20:ed:49:b2:c1:86:7c:29:92:74:1c:1f (ECDSA)
|_  256 18:cd:9d:08:a6:21:a8:b8:b6:f7:9f:8d:40:51:54:fb (ED25519)
80/tcp    open  http    nginx 1.18.0 (Ubuntu)
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-title: Input string was not in a correct format.
|_http-server-header: nginx/1.18.0 (Ubuntu)
88/tcp    open  http    nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
| http-robots.txt: 1 disallowed entry 
|_/
|_http-title: phpMyAdmin
9999/tcp  open  http    nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: Input string was not in a correct format.
| http-methods: 
|_  Potentially risky methods: TRACE
56423/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Site doesn't have a title (application/json;charset=utf-8).
|_http-server-header: Fulcrum-API Beta
```

A few things stand out immediately:

- **Port 4** - nginx, "Under Maintance" page, uses `index.php?page=` routing
- **Port 22** - standard OpenSSH on Ubuntu 20.04
- **Port 80 / 9999** - both throw `Server Error in '/' Application.` which is an ASP.NET-style error message, interesting on an nginx/Ubuntu front
- **Port 88** - phpMyAdmin (note: not Kerberos despite the default service name)
- **Port 56423** - `Fulcrum-API Beta`, returns JSON

The `Input string was not in a correct format.` and `Server Error in '/' Application.` messages on 80/9999 are classic .NET exceptions, so nginx is almost certainly reverse-proxying backend apps.

## Initial Web App Enumeration

### 4/tcp - "Under Maintance"

```
<h1>Under Maintance</h1><p>Please <a href="http://10.129.136.254:4/index.php?page=home">try again</a> later.</p>
```

The `try again` button redirects to `http://10.129.136.254:4/index.php?page=home`. A `page=` parameter that loads `home` is a textbook LFI/RFI candidate, so this is the one to come back to.

### 80/tcp - nginx/1.18.0

`Server Error in '/' Application.` - .NET error which is also seen on port 9999.

### 88/tcp - phpMyAdmin

phpMyAdmin login page. Default `root:<BLANK>` returns `Login without a password is forbidden by configuration`. Variations like `root:root`, `admin:admin` all return `mysqli_real_connect(): (HY000/2002): No such file or directory`, which suggests the MySQL socket/backend isn't reachable the way phpMyAdmin expects, park this for later.

### 9999/tcp - nginx/1.18.0

`Server Error in '/' Application.` - same as port 80.

### 56423/tcp - Fulcrum-API Beta

```
{"Heartbeat":{"Ping":"Pong"}}
```

A beta API with a heartbeat endpoint. Worth fuzzing for additional routes once I have more context.

## 4/tcp - Deeper Enumeration

This is the port that sparked the most interest. Since the app is `index.php`, I ran a feroxbuster scan looking specifically for the PHP extension:

```
┌──(corey㉿kali)-[~/htb/Fulcrum]
└─$ feroxbuster -u http://10.129.136.254:4/ -w /opt/wordlists/subdomains/subdomains-top1million-110000.txt -x php -t 100

200      GET        1l        6w      113c http://10.129.136.254:4/index.php
200      GET        1l        6w      113c http://10.129.136.254:4/
200      GET        1l        6w       54c http://10.129.136.254:4/upload.php
200      GET       13l       27w      312c http://10.129.136.254:4/home.php
```

Two new pages: `upload.php` and `home.php`.

`home.php` presents a file upload form. Uploading a test image sends a POST to `upload.php`:

```
POST /upload.php HTTP/1.1
Host: 10.129.136.254:4
Content-Length: 4582
Content-Type: multipart/form-data; boundary=----WebKitFormBoundarynwf90OHDs3f7DBJ4
...

------WebKitFormBoundarynwf90OHDs3f7DBJ4
Content-Disposition: form-data; name="fileToUpload"; filename="pfp.png"
Content-Type: image/png

<...image contents...>

------WebKitFormBoundarynwf90OHDs3f7DBJ4
Content-Disposition: form-data; name="submit"

Upload Image
------WebKitFormBoundarynwf90OHDs3f7DBJ4--
```


## Foothold


## Lessons Learned

