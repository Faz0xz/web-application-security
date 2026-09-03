# Stored XSS into onclick event with angle brackets and double quotes HTML-encoded and single quotes and backslash escaped

**Source:** PortSwigger Web Security Academy  
**Category:** Reflected XSS  
**Difficulty:** Practitioner  

## Objective

This lab contains a stored cross-site scripting vulnerability in the comment functionality. To solve this lab, submit a comment that calls the `alert()` function when the comment author name is clicked. 

Initial Analysis

The application features a blog post where users can submit comments. The comment author's name is rendered as a clickable element wired to an onclick event handler that embeds the author's website URL.

This means the input is placed inside a JavaScript context (an inline event handler), not simply reflected as HTML text.

The lab description states that:

Character	Handling
< >	HTML-encoded
"	HTML-encoded
'	Escaped
\	Escaped

Because the input lands inside a JavaScript string within the onclick attribute, the first step is to determine how the application handles characters that could be used to break out of that string.

Testing the Input

Since single quotes are escaped, a direct attempt to terminate the JavaScript string with a literal quote fails. For example:

'-alert()-'

The single quotes here get escaped by the application before being reflected, so the string context is never broken.

Backslashes are escaped as well, so the classic technique of using a backslash to interfere with the application's escaping logic (as in the reflected XSS-into-JS-string case) doesn't work here either.

The key insight is that the input passes through two separate parsing stages:

HTML parsing — the browser first parses the page as HTML, decoding any HTML entities it encounters.
JavaScript parsing — the decoded attribute value is then parsed and executed as JavaScript when the onclick fires.

The application's filters only account for literal characters — not for characters supplied as HTML entities, which get decoded by the HTML parser before the JavaScript parser ever sees them.

Bypassing the Escaping

Instead of submitting a literal single quote, I submitted its HTML entity equivalent:

&apos;

Because this is an HTML entity rather than a literal ' character, the application's string-escaping filter doesn't recognize or touch it. The browser's HTML parser decodes &apos; into an actual ' character after the filter has already run — meaning the quote reappears intact by the time the JavaScript parser processes the onclick attribute.

This effectively smuggles a real single quote through the filter, allowing me to break out of the JavaScript string.

Payload

The final payload was submitted in the comment author's website field:

http://post?&apos;-alert()-&apos;

Breakdown:

Segment	Purpose
&apos;	HTML-encoded single quote — survives the filter, decodes to ' before JS execution, closing the original string
-alert()-	The injected JavaScript expression
&apos;	A second encoded quote, reopening a string literal so the JavaScript following the injection remains syntactically valid

The - (minus) operators are used instead of ; to keep everything within a single valid JavaScript expression, since the onclick attribute's value is parsed as an expression rather than a full statement block. Using subtraction is a common trick to concatenate an injected call with surrounding string literals without breaking syntax.

Resulting Behavior

Conceptually, the vulnerable markup looks like:

html
<a onclick="viewProfile('INJECTION_POINT')">author</a>

After the browser decodes the HTML entities and the JavaScript is evaluated, it effectively becomes:

javascript
viewProfile(''-alert()-'')

Which the JavaScript engine evaluates as: close the empty string, subtract the result of alert() (which pops the alert first), then subtract another empty string — resulting in NaN, but not before alert() has already executed as a side effect.

Root Cause

The vulnerability stems from unsafe insertion of user-controlled data into an inline JavaScript event handler (onclick). Although the application attempted to sanitize input by escaping certain literal characters, this escaping did not account for the fact that the data passes through multiple parsing contexts before execution:

User input
   │
   ▼
HTML parsing (decodes HTML entities)
   │
   ▼
onclick attribute value
   │
   ▼
JavaScript parsing
   │
   ▼
Injected JavaScript execution

Because the escaping filter only operated on literal characters and not on their HTML-encoded equivalents, an attacker could smuggle a functional single quote through the filter by encoding it, letting the HTML parser "decode away" the protection before the JavaScript parser ever saw it.

Remediation
Avoid placing user-controlled data directly inside inline JavaScript event handlers such as onclick.
Instead of:
html
  <a onclick="viewProfile('...userInput...')">

use JavaScript event listeners that keep user-controlled data separate from executable code:

javascript
  element.addEventListener('click', () => viewProfile(safeUserValue));
Apply encoding that accounts for all parsing stages the data will pass through, not just the outermost one — a filter must be aware that HTML entity decoding happens before JavaScript execution.
Prefer setting data via safe DOM properties (textContent, dataset) and reading it back in the handler, rather than string-interpolating it into inline script.
Deploy a restrictive Content Security Policy that disallows inline event handlers and inline scripts.
Conclusion

This lab demonstrated that character-level escaping is insufficient when input traverses multiple parsers before execution. The application correctly blocked literal single quotes and backslashes, but failed to account for HTML-entity-encoded equivalents that would later be decoded by the browser's HTML parser — reintroducing the very characters the filter was meant to block, just after the filter had already run. The lesson: defenses must be applied with awareness of the full parsing pipeline, not just the immediate context where input first appears.
