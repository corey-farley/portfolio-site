---
date: 2026-10-02
title: "WIP - Host Header SSRF Explained - WIP"
categories: ["Web App Pentesting"]
tags: ["SSRF", "Host Header", "Web", "PortSwigger", "Burp Suite"]
author: "Corey Farley"
summary: "How the Host header becomes an SSRF primitive, how front-end routing works, why trusting the Host header breaks the trust boundary between proxy and back-end, and a full walkthrough of pivoting into an internal admin panel to delete a user. Includes a quick tour of the other Host header attack paths and the bypass tricks worth having on hand when the header is validated."
showToc: true
cover:
 image: cover.png
---

Routing-based SSRF is one of those bugs that feels almost too simple to be real the first time you land it. You change a single header, and suddenly the front-end is making requests into the internal network on your behalf. No clever parser abuse, no encoding tricks, just the server trusting a value it should never have trusted.

This write-up covers what the attack actually is, why the architecture that enables it is so common, and then walks the full exploitation path on the **PortSwigger Lab: Routing-based SSRF**, where we pivot to an internal admin panel and delete a user. At the end I've thrown in the other Host header attack paths and the bypass tricks worth knowing, because the plain version in the lab is the easy case.

## What is Host header SSRF?

Most real applications don't sit on a single server. There's a front-end — a reverse proxy, load balancer, or CDN — that receives your request and forwards it to a back-end somewhere behind it. The question is: *how does the front-end decide where to forward?*

In a lot of setups, it decides based on the `Host` header. That's virtual host routing, and it's completely normal — one proxy fronting many back-end services, picking the destination from the hostname you asked for.

The vulnerability shows up when the front-end forwards to **whatever** host you put in that header, with no allowlist. Swap the legitimate hostname for an internal IP and the proxy dutifully connects to it and hands you back the response. You've turned the front-end into an open proxy into its own internal network. That's routing-based SSRF.

WIP WIP WIP WIP WIP WIP WIP