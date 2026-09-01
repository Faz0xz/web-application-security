# Reflected XSS into a JavaScript String with Single Quote and Backslash Escaped

**Source:** PortSwigger Web Security Academy  
**Category:** Reflected XSS  
**Difficulty:** Practitioner  

## Objective

This lab contains a reflected cross-site scripting vulnerability in the search query tracking functionality. The reflection occurs inside a JavaScript string with single quotes and backslashes escaped. To solve this lab, perform a cross-site scripting attack that breaks out of the JavaScript string and calls the alert function. 

## Source

The `search` GET parameter — reflected into an inline `<script>` block.

## Initial Reconnaissance — Testing Special Characters

Rather than immediately trying payloads, I first mapped how the application
handled special characters:

| Input | Reflected as | Result |
|---|---|---|
| `'` | `\'` | Escaped — cannot terminate string directly |
| `\` | `\\` | Escaped — cannot use as escape prefix |
| `\\'` | `\\\\'` | Both escaped — combination also blocked |

The application consistently escaped both characters. The direct approach of
closing the JavaScript string with a single quote was not viable.

## Finding the JavaScript Context

I was initially focused on the HTML reflection and tried to escape by using different sequences of backslashes and quotes to insert an html tag which would trigger an alert which yielded no progress. After which I checked the network tab to see if any external js files were being loaded which did the filtering which bore no fruit either and a script embeded within the image which was taking the input and working upon it. The relevant JavaScript was visible in an inline `<script>` block:

```javascript
var searchTerms = 'USER_INPUT';
document.write('<img src="/resources/images/tracker.gif?searchTerms='
  + encodeURIComponent(searchTerms) + '">');
```

The injection point was clear: input was placed directly inside a single-quoted JavaScript string assigned to `searchTerms` which was using a document.write function which executes all tags within the function as js code which is where the security vulnerability lies.

## Changing Approach — Two Parsers, Not One

I then tried to escape the javascript function so that I could insert another tag which would execute, but that didn't bore any results. Since the JavaScript-layer escaping was also pointless, I reconsidered the problem from a different angle: the browser doesn't use one parser to process a page. It uses two in sequence.

When the browser loads a page containing a `<script>` element:

```
HTML parser runs first
      ↓
Finds <script> tag
      ↓
Reads content until it encounters </script>
      ↓
Hands that content to the JavaScript parser
```

The application was escaping characters relevant to the **JavaScript parser** specifically `'` and `\`, which matter for string literal syntax. But the HTML parser uses completely different rules. It doesn't know or care about JavaScript string delimiters. It only looks for the literal sequence `</script>` to decide where the script element ends.

The application escaped nothing relevant to the HTML parser.

## Exploit

```html
</script><script>alert(1)</script>
```

When reflected inside the page, the structure becomes:

```html
<script>
  var searchTerms = '</script><script>alert(1)</script>';
  document.write(...)
</script>
```

The HTML parser processes this as:

```html
<!-- First script element — terminated by injected </script> -->
<script>
  var searchTerms = '
</script>

<!-- Second script element — injected fresh block -->
<script>alert(1)</script>

<!-- Developer's closing tag — now orphaned, ignored -->
';
  document.write(...)
</script>
```

The HTML parser terminates the original `<script>` element at the injected `</script>`. The remaining content of the JavaScript string — the `'` and the `document.write` line — is now outside any script context and is treated as plain HTML text, which the browser ignores. The injected `<script>alert(1)</script>` is parsed as a fresh, independent script block. `alert(1)` executes.

## Why the JavaScript Escaping Was Irrelevant

The application protected the JavaScript string context by escaping `'` and `\`. This would be sufficient if the JavaScript parser were the only parser involved. But the HTML parser runs first and never sees JavaScript string syntax it only looks for `</script>` as a tag boundary.

By targeting the HTML parser instead of the JavaScript parser, the JavaScript-level escaping was entirely bypassed. The two parsers were operating on different rules, and the defense was only covering one of them.

## Parser Interaction — Visual Summary

```
Browser receives HTTP response
      ↓
HTML parser reads document top to bottom
      ↓
Finds <script> → begins collecting script content
      ↓
Finds </script> (the injected one) → ends script block
      ↓
Passes incomplete JS to JavaScript parser (fails silently)
      ↓
HTML parser continues → finds <script>alert(1)</script>
      ↓
JavaScript parser executes alert(1)
```

Remediation

The primary remediation is to avoid using document.write() with user-controlled data. document.write() writes directly into the HTML document and can cause user-controlled content to be interpreted as markup. A safer approach is to create DOM elements explicitly and assign untrusted values using APIs such as textContent or safely constrained attribute setters.

For example, instead of:

document.write('<img src="/resources/images/tracker.gif?searchTerms=' + encodeURIComponent(searchTerms) + '">');

the application could create the element programmatically:

const img = document.createElement('img');
img.src = '/resources/images/tracker.gif?searchTerms=' + encodeURIComponent(searchTerms);
document.body.appendChild(img);

The application should also implement proper context aware output encoding. Data being inserted into JavaScript, HTML, HTML attributes, CSS, or URLs must be encoded according to the context in which it is being used. Escaping a few characters such as single quotes and backslashes is not sufficient protection against XSS.

A Content Security Policy (CSP) should also be implemented to reduce the impact of XSS vulnerabilities. In particular, the application should avoid allowing unrestricted inline scripts and should use nonces or hashes for scripts that must be executed.

User-controlled input should be treated as untrusted throughout the application. Input validation can be used to enforce expected formats, but it should not be relied upon as the primary XSS defense.

The application should also avoid dynamically constructing HTML with untrusted data. Where possible, DOM APIs such as createElement(), textContent, and controlled property assignments should be preferred over functions that interpret strings as HTML.

Finally, security testing should cover different parser contexts rather than testing only for common payloads. In this case, single quotes and backslashes were correctly escaped, but the application remained vulnerable because the HTML parser could interpret </script> and terminate the surrounding script element.
