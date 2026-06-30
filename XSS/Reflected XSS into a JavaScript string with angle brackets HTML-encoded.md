# Reflected XSS into a JavaScript string with angle brackets HTML-encoded

**Source:** PortSwigger Web Security Academy
**Category:** Reflected XSS
**Difficulty:** Apprentice

## Objective

The lab's search functionality reflects user input inside a JavaScript string.
Angle brackets are HTML-encoded server-side. Goal: break out of the string and
trigger `alert()`.

## Initial Reconnaissance

- Input source: `search` GET parameter.
- The search term is reflected twice in the response — once in the page body,
  once inside an inline `<script>` block.

## Reflection #1 — page body

```html
<h1>0 search results for '&quot; &lt;Test'</h1>
```

Angle brackets are encoded and the value lands inside a text node. No parser
confusion possible here — not exploitable because no way to create a new html tag such as an <img> tag an execute alert() function or even if we escape the tag the application will not parse js code.

## Reflection #2 — inline script

```javascript
<script>
  var searchTerms = '" <Test';
  document.write(
      '<img src="/resources/images/tracker.gif?searchTerms=' +
      encodeURIComponent(searchTerms) +
      '">'
  );
</script>
```

This is the live reflection. The parser context here is JavaScript, not HTML —
which matters, because the server's encoding defends the HTML context
(reflection #1) but does nothing for this one. Angle-bracket encoding is
irrelevant inside a JS string literal.

## Sink

`var searchTerms = 'USER_INPUT';` — a JavaScript string literal assignment.

## Data Flow

```
search parameter
      ↓
server reflects value into inline <script> block
      ↓
browser's HTML parser hands script block to JS engine as source text
      ↓
JS engine parses and executes the line
```

## Current Context

```javascript
var searchTerms = 'HERE';
```

Everything before and after the injection point is fixed, developer-written
code — `var searchTerms = '` precedes it, `';` follows it. Both pieces still
get concatenated into the final source regardless of what's injected.

## Investigation

**Test 1:** `TEST' alert();`

Resulting source:
```javascript
var searchTerms = 'TEST' alert();';
```
Did not fire. The single quote closed the string successfully, but the
result is not valid JavaScript — `'TEST'` and `alert()` sit next to each
other with no statement boundary between them, and the trailing `';` is
left dangling as an unterminated string. Closing the string is necessary
but not sufficient; the statement itself was never properly terminated.

**Test 2:** `TEST'; alert();//'`

Reasoning: add an explicit statement terminator (`;`) after closing the
string, then neutralize everything the application appends afterward using
a single-line comment (`//`).

Resulting source:
```javascript
var searchTerms = 'TEST'; alert();//';
```
This fired `alert()` successfully.

**Side investigation — false lead:** an early payload attempt showed `&apos;`
in the rendered view-source instead of a literal `'`. This looked like the
quote itself was being encoded, which would have invalidated the whole
approach. Checking the raw HTTP response body (not browser view-source
rendering) confirmed the quote was not actually encoded — view-source had
re-escaped it for display only. Lesson: verify payload behavior against raw
response bytes, not the browser's rendered representation of them.

## Final Payload Analysis

```
TEST'; alert();//
```

| Fragment | Role |
|---|---|
| `'` | Closes the developer's open string literal |
| `;` | Terminates that statement explicitly, resetting parser state to "ready for new statement" |
| `alert();` | Executes as an independent statement, outside any string |
| `//` | Single-line comment — neutralizes the developer's trailing `';` so it never reaches the parser as code |

## Browser Execution Flow

```
HTTP Response
      ↓
HTML Parser (builds DOM, hands <script> contents to JS engine as text)
      ↓
JS Engine parses line: closes string → terminates statement → executes alert() → comments out remainder
      ↓
alert() fires
```

## Lessons Learned

- Server-side encoding is context-specific. Encoding angle brackets defends
  the HTML parser context; it does nothing for a JavaScript string context
  reflected elsewhere in the same response. The same input can be safe in
  one location and exploitable in another, on the same page.
- Closing a string literal and terminating the enclosing statement are two
  separate requirements. A payload can successfully escape the string and
  still fail to execute if the statement boundary isn't also satisfied.
- A trailing comment isn't just "cleanup" — it's what allows arbitrary
  leftover developer syntax to be ignored safely, without needing to predict
  or replicate it.
- Browser dev tools' view-source rendering can re-escape characters for
  display. Payload verification should happen against raw response bytes
  (e.g. via Burp), not rendered view-source output.

## Professional Takeaways

- Map every reflection of user input before deciding which one is
  exploitable — the first one found is not necessarily the right one.
- Identify the parser context precisely before attempting to craft a
  payload; HTML-safe input is not automatically JavaScript-safe.
- When breaking out of a scripting context, account for both what precedes
  and what follows the injection point in the original source.
