# Reflected XSS into a JavaScript string with single quote and backslash escaped

**Source:** PortSwigger Web Security Academy  
**Category:** Reflected XSS  
**Difficulty:** Practitioner  

## Objective

This lab contains a reflected cross-site scripting vulnerability in the search query tracking functionality where angle brackets and double are HTML encoded and single quotes are escaped.

To solve this lab, perform a cross-site scripting attack that breaks out of the JavaScript string and calls the alert function. 

| Character | Handling |
|---|---|
| `<` `>` | HTML-encoded |
| `"` | HTML-encoded |
| `'` | Escaped with a backslash (`\'`) |
| `\` | Not encoded or escaped |

The objective was to break out of the JavaScript string context and trigger `alert()`.

## Initial Testing

I began by submitting individual special characters to observe how each was handled.

Submitting a single quote:

```
'
```

was reflected as:

```
\'
```

Since the application automatically escapes any single quote I submit, injecting a raw `'` alone isn't enough to terminate the string.

Next, I tested a backslash on its own and found it was reflected **unmodified**. This was the key gap in the filter — the backslash character isn't escaped or removed at all.

## Understanding the Escaping Logic

Because the application blindly prepends a backslash to every `'` I send, I can manipulate that behavior by supplying my own leading backslash:

**Input:**
```
\'
```

**Application escapes the quote, producing:**
```
\\'
```

The JavaScript parser now sees two consecutive backslashes followed by a quote. It interprets `\\` as a single escaped (literal) backslash character — not as an escape sequence for the quote that follows:

```
\\   '
 ↑    ↑
one literal   this quote is now
backslash     unescaped and closes
              the string
```

In other words, my injected backslash "consumes" the escaping backslash the application adds, leaving the actual single quote free to close the JavaScript string literal.

## Building the Payload

Final payload:

```
\';alert(1)//
```

Breakdown:

| Segment | Purpose |
|---|---|
| `\'` | My backslash pairs with the application's auto-inserted backslash, so the resulting `\\` is treated as one literal backslash — leaving the quote free to close the string |
| `;` | Terminates the original JavaScript statement |
| `alert(1)` | The injected payload |
| `//` | Comments out the remainder of the original line, neutralizing any trailing syntax added by the app |

## Resulting JavaScript

Before injection, the vulnerable line looks something like:

```javascript
var searchTerms = 'INJECTION_POINT';
```

After injection, it effectively becomes:

```javascript
var searchTerms = '\\';alert(1)//';
```

Which the parser resolves as three logical statements:

```javascript
var searchTerms = '\\';   // string closed cleanly
alert(1)                  // injected payload executes
//';                      // rest of the line commented out
```

`alert(1)` fires successfully, confirming the XSS.

## Remediation

- Never construct executable JavaScript by directly concatenating user-controlled input into `<script>` blocks.
- Apply **context-aware output encoding** — specifically JavaScript string escaping (e.g., escaping backslashes *before* quotes, not after) rather than naive single-character substitution.
- Prefer safe APIs (`textContent`, `dataset`, JSON embedding via `JSON.stringify` with proper escaping) over building raw JS/HTML strings.
- Deploy a restrictive **Content Security Policy** (e.g., disallowing inline scripts) as defense in depth.
- Treat all user input as untrusted, regardless of where in the response it's reflected.

## Conclusion

This lab highlighted that escaping routines can introduce new vulnerabilities if they don't account for the escape character itself. The application correctly escaped single quotes but left backslashes completely untouched — and since backslash is the escape character in JavaScript strings, that omission was fatal. By prepending my own backslash, I turned the application's defensive `\'` insertion against itself, canceled out the escaping, broke out of the string, and executed arbitrary JavaScript. The main lesson: understanding *why* a filter behaves the way it does is far more effective than blindly trying stock XSS payloads.
