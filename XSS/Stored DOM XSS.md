# Stored DOM XSS

**Source:** PortSwigger Web Security Academy  
**Category:** Stored DOM XSS  
**Difficulty:** Practitioner  

## Objective

This lab demonstrates a stored DOM vulnerability in the blog comment functionality. To solve this lab, exploit this vulnerability to call the `alert()` function. 
The blog comment functionality stores user input and later renders it via client-side JavaScript. Goal: exploit a broken sanitisation function to inject an HTML payload that executes `alert()`.

## Source

Two fields in the blog comment form:
- `comment` body field
- `name` (author) field

Email and website fields are validated both client-side and server-side — intercepting the request and bypassing client-side controls confirmed these were not injectable.

## Initial Reconnaissance

Test input: `<Test3"')};`

Result: input was reflected back inside a `<p>` tag with no visible HTML
encoding. Nothing obviously broken from the surface — the input appeared as
plain text in the DOM. This confirmed input was being stored and rendered,
but the rendering mechanism wasn't yet clear.

<img width="936" height="718" alt="Screenshot 2026-07-07 142926" src="https://github.com/user-attachments/assets/776216bd-649b-4c74-90d3-284bdee8e332" />
<img width="911" height="112" alt="Screenshot 2026-07-07 142939" src="https://github.com/user-attachments/assets/474d07a4-aec8-4c62-86c5-a23def17de49" />
<img width="531" height="172" alt="Screenshot 2026-07-04 191211" src="https://github.com/user-attachments/assets/ff0625cf-16bb-460c-bea4-9466fc6ac654" />


Early attempt: submitted `<img src="0" onerror=alert()>` as a comment.
<img width="836" height="150" alt="Screenshot 2026-07-04 191555" src="https://github.com/user-attachments/assets/a578e755-9d3b-4f5c-b457-c422c0ded2d0" />
<img width="707" height="158" alt="Screenshot 2026-07-07 170142" src="https://github.com/user-attachments/assets/0a914855-2564-418c-8ba4-bc68f31ef5da" />

Result: did not fire. The img tag was not parsed as HTML — it was treated as a text string inside the `<p>` element. The browser never created an img element, so the `onerror` handler never triggered.

This meant something client-side was processing the input before it reached
the DOM, stripping or encoding the tags before `innerHTML` could parse them.

## Finding the Sink

Inspecting page source revealed an external JavaScript file being loaded. Opening it in DevTools revealed two relevant functions:

**`escapeHTML(html)`** — intended to sanitise input before rendering:

```javascript
function escapeHTML(html) {
    return html.replace('<', '&lt;').replace('>', '&gt;');
}
```

**`displayComments()`** — renders stored comments into the DOM using `innerHTML`:

```javascript
let newInnerHtml = firstPElement.innerHTML + escapeHTML(comment.author)
firstPElement.innerHTML = newInnerHtml
```

```javascript
commentBodyPElement.innerHTML = escapeHTML(comment.body);
```

The sink is `innerHTML`. User input flows through `escapeHTML()` first, then gets written into the DOM via `innerHTML`, which parses its content as HTML. If anything survives `escapeHTML()` with intact angle brackets, the browser
will render it as a real HTML element.

## Data Flow

```
comment body / author field (stored in database)
      ↓
page load triggers loadComments() via XHR
      ↓
JSON response parsed — comment fields extracted
      ↓
escapeHTML() attempts to sanitise input
      ↓
sanitised string assigned to innerHTML (sink)
      ↓
browser parses string as HTML — renders any surviving tags
```

## Identifying the Vulnerability with DevTools

Set a breakpoint on the `escapeHTML()` function in the Sources panel. Reloaded the page — execution paused at the breakpoint and the Scope panel revealed the live value of `html` at that moment:

```
html: "omg bob did you see this post"
```

This confirmed the comment body is passed directly into `escapeHTML()` as a string — exactly as submitted, before any encoding. The breakpoint made the data flow visible without needing to fully read the JS.

## The Vulnerability

`escapeHTML()` uses JavaScript's `.replace()` method:

```javascript
html.replace('<', '&lt;').replace('>', '&gt;')
```

<img width="510" height="276" alt="Screenshot 2026-07-07 143153" src="https://github.com/user-attachments/assets/e261e5a4-7bbc-4941-8277-7a8874283e07" />


`.replace()` with a plain string argument only replaces the **first** occurrence. A correct implementation would use a regular expression with the global flag:

```javascript
html.replace(/</g, '&lt;').replace(/>/g, '&gt;')
```

Without the `g` flag, any input containing more than one `<` or `>` will have only the first instance encoded — every subsequent tag passes through unmodified and gets parsed by `innerHTML` as real HTML.

## Failed Attempts and What They Revealed

| Input | Result | Reason |
|---|---|---|
| `<img src="0" onerror=alert()>` | No alert — rendered as text | First `<` encoded, `>` encoded — entire tag neutralised |
| `<Test3"')};` in comment | Reflected as plain text | Appeared inside `<p>` — not parsed as HTML, encoding caught the tags |

The pattern: a single tag always gets fully caught. The first `<` is encoded,the first `>` is encoded — the whole tag is neutralised.

## Payload Construction

The fix was to lead with a decoy pair of angle brackets that `escapeHTML()` consumes, exhausting its single replacement, then follow with the real payload whose angle brackets pass through unmodified:

```
<><img src=0 onerror=alert()>
```

**What escapeHTML() does to this:**

- First `<` → encoded to `&lt;` (replacement consumed)
- First `>` → encoded to `&gt;` (replacement consumed)
- Remaining `<img src=0 onerror=alert()>` → passes through untouched

**What innerHTML sees:**

```html
&lt;&gt;<img src=0 onerror=alert()>
```

The `&lt;&gt;` renders as harmless literal text `<>`. The `<img>` tag is
parsed as a real HTML element — `src=0` fails to load, `onerror` fires,
`alert()` executes.

## Final Payload Breakdown

| Fragment | Role |
|---|---|
| `<>` | Decoy — consumed by escapeHTML()'s single replacement for `<` and `>` |
| `<img src=0` | Real payload — passes through unencoded, parsed as HTML element by innerHTML |
| `onerror=alert()>` | Event handler fires when image fails to load — executes alert() |

## Browser Execution Flow

```
Comment submitted with payload
      ↓
Stored in database
      ↓
Page load: loadComments() fetches comments via XHR
      ↓
displayComments() processes each comment
      ↓
escapeHTML() encodes first <> pair only — remainder passes through
      ↓
innerHTML receives string containing real <img> tag
      ↓
Browser parses <img> — creates image element in DOM
      ↓
Image load fails (src=0) — onerror fires
      ↓
alert() executes
```

## Lessons Learned

- Surface HTML encoding does not always mean the input is safe — check *how* the encoding is implemented, not just *that* it    exists.
- `.replace()` with a string argument only replaces the first match. Correct sanitisation requires regex with the global `/g` flag.
- The sink is `innerHTML` — `escapeHTML()` is the broken defence in front of it. Both are part of the same vulnerability chain.
- Setting a breakpoint in DevTools and reading the live Scope panel is a practical alternative to fully understanding JavaScript code — the browser shows you exactly what value is flowing through a function at
  runtime.
- Always check every input field on a form, not just the obvious one. Both the comment body and name field were vulnerable here.
- When a page loads comments or user-generated content, always check for an external JS file handling the rendering — the vulnerability is frequently in client-side display logic, not server-side output. Tho most js files are obfuscated in modern applications.
- Inspect sanitisation functions carefully. A function named `escapeHTML` or `sanitise` provides false confidence — verify the implementation.
- DevTools breakpoints are a first-class reconnaissance tool for reading data flow through client-side JS without needing to be a developer.
