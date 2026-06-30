# Reflected XSS into HTML Context (Nothing Encoded)

## Objective

Understand how reflected Cross-Site Scripting (XSS) occurs when user-controlled input is inserted directly into an HTML document without output encoding.

---

## Lab Information

**Category:** Cross-Site Scripting (XSS)

**Type:** Reflected XSS

**Difficulty:** Apprentice

**Source:** PortSwigger Academy

---

# Reconnaissance

The application provides a search functionality where user input is reflected back to the page.

My first objective was to determine:

- Where my input originates (Source)
- Where my input is reflected
- Whether it is reflected inside HTML, JavaScript, or an HTML attribute

---

# Source

The attacker controls the `search` parameter through the search form.

```
User Input
        │
        ▼
Search Parameter
```

---

# Reflection Analysis

I first searched for a simple marker.

```
abz
```

The application reflected my input inside the following HTML.

```html
<h1>
0 search results for 'abz'
</h1>
```

I observed that:

- My input was reflected only once.
- The reflection occurred inside an `<h1>` element.
- No output encoding was applied.

---

# Sink

The sink is the HTML document itself.

The browser inserts my input directly into the page.

```html
<h1>
USER INPUT
</h1>
```

---

# Parser

**HTML Parser**

This was the first important observation.

The browser is currently parsing HTML.

It is **not** parsing JavaScript.

---

# Current Context

My payload is inserted inside normal HTML.

```html
<h1>
HERE
</h1>
```

Since the parser expects HTML, any payload I use must also be valid HTML.

---

# Investigation

## Test 1

Input

```
abz
```

Observation

The browser rendered the text normally while appending a single quote at the end which was not given in the input.

Conclusion

The application reflects user input directly into the HTML response.

---

## Test 2

Input

```
'
```

Observation

The application surrounds my input with a quote.

```html
<h1>
0 search results for '''
</h1>
```

This indicated that my input was surrounded by existing HTML content.

---

## Hypothesis

Since my input is interpreted as HTML, I should be able to inject a new HTML element.

The HTML parser does **not** execute raw JavaScript.

For example,

```html
alert()
```

would simply be rendered as text.

Instead, I needed to inject valid HTML that would eventually execute JavaScript.

---

## Exploit Reasoning

I considered two possible HTML elements.

### Option 1

```html
a' <script>alert()</script>
```

Reasoning

The HTML parser creates a `<script>` element.

The browser immediately executes the JavaScript contained inside it.

---

### Option 2

```html
a' <img src="0" onerror="alert()">
```

Reasoning

The HTML parser creates an image element.

The browser attempts to load the image.

Since the image does not exist, an error occurs.

The browser fires the `onerror` event, executing the JavaScript.

---

# Browser Execution Flow

```
HTTP Response
        │
        ▼
HTML Parser
        │
        ▼
Creates HTML Elements
        │
        ▼
<img> Element
        │
        ▼
Image Load Fails
        │
        ▼
onerror Event Fires
        │
        ▼
JavaScript Engine
        │
        ▼
alert()
```

---

# Root Cause

The application reflects attacker-controlled input directly into an HTML document without performing output encoding.

Because the browser interprets the injected data as HTML, an attacker can create arbitrary HTML elements capable of executing JavaScript.

---

# Lessons Learned

- Reflected XSS occurs immediately in the server response.
- Reflection alone does not imply XSS; the context determines exploitability.
- The HTML parser expects HTML, not JavaScript.
- Successful payloads must match the parser currently interpreting the input.
- Browser events (such as `onerror`) can be used to reach the JavaScript engine.
- Multiple payloads may work because they are valid HTML elements that eventually execute JavaScript.

---



I now ask:

> "What parser is reading my input, and what syntax does it expect?"
