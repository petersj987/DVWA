## Overview

| Field | Detail |
|---|---|
| **Vulnerability** | Reflected Cross-Site Scripting (XSS) |
| **Security Level** | Medium |
| **Method** | GET parameter injection via `name` field |

---

## What Changed from Low

At medium security, the developer added a single filter:

```php
echo 'Hello ' . str_replace('<script>', '', $_GET['name']);
```

Compared to low:

```php
echo 'Hello ' . $_GET['name'];  // Low — no filter at all
```

The intent is to block `<script>` tag injection. In practice, the filter is trivially bypassed.

---

## Filter Analysis

### What `str_replace()` does here

`str_replace('<script>', '', $input)` scans the input string for the exact substring `<script>` and replaces it with nothing (empty string). It does this once, on a hardcoded lowercase string.

### Why it fails

| Weakness | Detail |
|---|---|
| **Case sensitive** | Only matches lowercase `<script>`. Mixed/uppercase variants pass through untouched |
| **Single pass, non-recursive** | Strips once — nested payloads reconstruct after removal |
| **Tag-specific** | Entirely ignores event handler vectors like `onerror`, `onload`, `onmouseover` |

---

## Exploitation

### Bypass 1 — Case Variation

**Payload:**
```html
<Script>alert('XSS')</Script>
```

**Why it works:**  
`str_replace()` is case-sensitive. It looks for `<script>` (lowercase) only. `<Script>` does not match, so it passes through the filter untouched. The browser's HTML parser is case-insensitive and executes it normally.

**Result:** Alert box fires with `XSS`.

---

### Bypass 2 — Nested Tag Reconstruction

**Payload:**
```html
<sc<script>ript>alert('XSS')</sc<script>ript>
```

**Why it works:**  
`str_replace()` finds and removes the inner `<script>`, leaving behind:

```
<sc + ript> = <script>
```

The filter's own removal operation reconstructs the exact tag it was trying to block. This is a classic single-pass blind spot.

**Result:** Alert box fires with `XSS`.

---

### Bypass 3 — Event Handler (No `<script>` Tag)

**Payload:**
```html
<img src=x onerror=alert('XSS')>
```

**Why it works:**  
The filter only looks for `<script>`. This payload uses an `<img>` tag with an `onerror` event handler — no `<script>` involved. When the browser fails to load `src=x`, `onerror` fires and executes the JavaScript.

**Result:** Alert box fires with `XSS`.

---

## Bypass Summary

| Bypass | Payload | Technique |
|---|---|---|
| 1 | `<Script>alert('XSS')</Script>` | Case variation |
| 2 | `<sc<script>ript>alert('XSS')</sc<script>ript>` | Nested tag reconstruction |
| 3 | `<img src=x onerror=alert('XSS')>` | Event handler — no script tag |

All three confirmed executing on DVWA medium security.

---

## Root Cause

Blocklist-based filtering is fundamentally unreliable for security. A developer cannot enumerate every possible XSS vector. The correct approach is output encoding — transforming characters so the browser never interprets them as code.

---

## Remediation

| Fix | Implementation |
|---|---|
| **Output encoding** | `htmlspecialchars($input, ENT_QUOTES, 'UTF-8')` — converts `<`, `>`, `"`, `'` to HTML entities |
| **Avoid blocklists** | Stripping specific tags is always bypassable; encoding is not |
| **Content Security Policy** | CSP header as defence-in-depth to block inline script execution |

---
