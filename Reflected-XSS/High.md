## Overview

| Field | Detail |
|---|---|
| **Vulnerability** | Reflected Cross-Site Scripting (XSS) |
| **Method** | GET parameter injection via `name` field |
| **Exploitable** | No |

---

## Source Code

```php
<?php

if(!array_key_exists("name", $_GET) || $_GET['name'] == NULL || $_GET['name'] == ''){
    $isempty = true;
} else {
    echo '<pre>';
    echo 'Hello ' . htmlspecialchars($_GET['name']);  // Correct fix
    echo '</pre>';
}

?>
```

---

## What Changed from Medium

| Level | Filter | Approach |
|---|---|---|
| Low | None | No protection |
| Medium | `str_replace('<script>', '', $input)` | Blocklist — wrong philosophy |
| High | `htmlspecialchars($input)` | Output encoding — correct philosophy |

The developer replaced the blocklist approach entirely with `htmlspecialchars()`, which encodes output rather than trying to predict and block malicious input patterns.

---

## How `htmlspecialchars()` Works

`htmlspecialchars()` converts characters that have special meaning in HTML into their entity equivalents before they reach the browser's parser:

| Character | HTML Entity |
|---|---|
| `<` | `&lt;` |
| `>` | `&gt;` |
| `"` | `&quot;` |
| `'` | `&#039;` |
| `&` | `&amp;` |

### Example

**Submitted payload:**
```
<img src=x onerror=alert('XSS')>
```

**What the browser actually receives:**
```
Hello &lt;img src=x onerror=alert(&#039;XSS&#039;)&gt;
```

**What renders on screen:**
```
Hello <img src=x onerror=alert('XSS')>
```

The browser displays it as plain text. There is no tag, no event handler, no execution.

---

## Bypass Attempts

All standard XSS vectors were tested and confirmed non-executable:

| Payload | Technique | Result |
|---|---|---|
| `<script>alert('XSS')</script>` | Basic script tag | Rendered as text |
| `<Script>alert('XSS')</Script>` | Case variation | Rendered as text |
| `<sc<script>ript>alert('XSS')</sc<script>ript>` | Nested tag reconstruction | Rendered as text |
| `<img src=x onerror=alert('XSS')>` | Event handler injection | Rendered as text |
| `"onmouseover="alert('XSS')` | Attribute injection | `"` encoded — injection fails |

**Conclusion:** `htmlspecialchars()` neutralizes every vector that depends on injecting HTML syntax characters. The input field is not exploitable at this security level.

---

## Key Lesson — Encoding vs Blocklisting

This is the core takeaway across all three security levels:

**Blocklisting** (Medium) tries to enumerate and remove known bad patterns. It will always fail because:
- Attackers know more vectors than developers can list
- Filters can be bypassed through case variation, nesting, encoding, or alternative tags
- Every new HTML feature potentially introduces a new bypass

**Output encoding** (High) makes the question of "what is malicious?" irrelevant. It transforms characters so the browser is structurally incapable of interpreting the input as code, regardless of what that input contains.

---

## Remediation (Confirmed Implemented)

| Control | Detail |
|---|---|
| **Output encoding** | `htmlspecialchars($input, ENT_QUOTES, 'UTF-8')` — note: adding `ENT_QUOTES` and explicit charset is best practice over the bare call used here |
| **Content Security Policy** | Defence-in-depth — restricts inline script execution even if encoding is somehow bypassed |
| **Input validation** | Where applicable, whitelist expected character sets at the point of input |

> **Note:** The high security implementation uses `htmlspecialchars()` without `ENT_QUOTES` or an explicit charset. Best practice is `htmlspecialchars($input, ENT_QUOTES, 'UTF-8')` to also encode single quotes and avoid charset edge cases.

---
