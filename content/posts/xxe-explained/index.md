---
date: 2026-09-23
title: "WIP - XXE Explained - WIP"
categories: ["Web App Pentesting"]
tags: ["XXE", "Web", "SSRF", "PortSwigger", "OOB"]
author: "Corey Farley"
summary: "A full breakdown of XML External Entity (XXE) injection: what it is, the XML internals that make it possible, where the attack surface hides, and a few examples of attack paths. The examples include basic in-band file read, blind out-of-band (OOB) exfil via malicious external DTDs, SSRF against cloud metadata, and SVG file upload vectors."
showToc: true
cover:
 image: cover.png
---

XXE (XML External Entity injection) is one of those vulnerabilities that looks intimidating the first time you see it, but it's really not that bad once you understand it. Once you understand how XML entities actually resolve, the different variants (in-band, blind, out-of-band, error-based) it's the same core trick just wearing a different hat for data exfil.

This article walks through what XXE is, the XML internals behind it, where it shows up, and each of the main exploitation paths using the PortSwigger Academy labs as examples.

## What is XXE?

XML External Entity injection is a vulnerability that lets an attacker interfere with an application's processing of XML data. It occurs when an app parses XML input and the underlying parser is configured (usually by default) to resolve **external entities** which are references that pull in content from outside the document, including local files and network URLs.

When that's the case, you can:

- Read arbitrary files off the server's filesystem
- Force the server to make requests to internal systems (SSRF)
- Exfiltrate data blindly when nothing is reflected back
- And in some parsers, trigger denial of service

The root cause is almost always a parser that was never hardened. The XML spec supports external entities, most language libraries enable them out of the box (or historically did), and they're often left on because developers forgot to turn them off.

WIP WIP WIP WIP WIP WIP WIP