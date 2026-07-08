# Reflected XSS into HTML Context with All Tags Blocked Except Custom Ones

**Source:** PortSwigger Web Security Academy  
**Category:** Reflected XSS  
**Difficulty:** Practitioner  

## Objective

To solve the lab, perform a cross-site scripting attack that injects a custom tag and automatically alerts document.cookie.  This lab blocks all HTML tags except custom ones. The search functionality reflects input as HTML but a WAF blocks all standard HTML tags. Goal: inject a custom HTML tag that automatically triggers `alert(document.cookie)` in a victim's browser session.

## Source

The `search` GET parameter user input submitted via the search form, reflected directly into the HTML response.

## Initial Reconnaissance

Test input: `<h1>hello</h1>`

Result: WAF returned JSON response — `"tag not allowed"`. Every standard HTML tag tested produced the same block. The WAF is filtering on known tag names, not on angle brackets or attributes themselves.

Test input: `<img src="0" onerror=alert()>`

Result: same — blocked.

Conclusion: the WAF maintains a blocklist of standard HTML tags. Attributes are not being filtered independently of the tag they belong to. As the lab description says we must use custom tags.

## Key Discovery — Custom HTML Tags

Custom HTML tags are valid in browsers and are not on any standard blocklist.
Custom tags follow specific rules:
- Must contain a hyphen (`-`)
- All characters must be lowercase
- Numbers are permitted

The WAF has no entry for custom tag names they pass through unchecked.

## Data Flow

```
search parameter
      ↓
GET request to server
      ↓
WAF checks tag name against blocklist — custom tags pass
      ↓
input reflected into HTML response
      ↓
browser parses custom tag as a real DOM element
      ↓
attributes on the element are processed by the browser
```

## Sink

The search results page reflects input directly into the HTML body. The browser parses the reflected content as HTML, custom tags included making any event handler attributes on those tags executable.

## Building the Payload

**Attempt 1 — manual trigger:**

```html
<custom-el onmouseover='alert()'>hello</custom-el>
```

Result: alert fired on mouseover. Confirmed the WAF does not block attributes, only tag names. But the lab requires automatic execution — no user interaction.

**Attempt 2 — onfocus:**

`onfocus` fires when an element receives browser focus. Replaced `onmouseover` with `onfocus`, but focus requires the element to actually receive focus. Custom elements are not focusable by default.

**Adding tabindex:**

`tabindex="1"` makes any element focusable — the browser can now direct focus to it via keyboard navigation or programmatically.

**Adding autofocus:**

`autofocus` tells the browser to automatically focus the element on page load, no user interaction required.

**Combined payload:**

```html
<my-cus onfocus="alert(document.cookie)" tabindex="0" autofocus>Hello</my-cus>
```

Tested in the lab directly — alert fires automatically on page load.

## Why the Lab Required the Exploit Server

Testing the payload in your own browser proves the injection works. But
reflected XSS only has real-world impact when it executes in a *victim's*
session — stealing their cookie, not yours.

The exploit server simulates delivering a malicious URL to a victim. Their
browser loads the crafted URL, the payload executes in their session, their
`document.cookie` is exposed. This is the actual attack — finding the
injection point is step one, demonstrating victim-session impact is step two.

This mirrors real bug bounty and pentest reporting requirements: a valid XSS
proof-of-concept must demonstrate exploitability against a victim session, not
just self-execution.

## Final Exploit Delivery

Payload delivered via exploit server:

```html
<script>
location = 'https://TARGET.web-security-academy.net/?search=%3Cmy-cus+onfocus%3D%22alert(document.cookie)%22+tabindex%3D%221%22+autofocus%3EHello%3C%2Fmy-cus%3E'
</script>
```

The `location` assignment redirects the victim's browser to the crafted URL.
The URL-encoded search parameter injects the custom tag. The victim's browser
parses it, focuses the element, fires `onfocus`, executes `alert(document.cookie)`.

## The `#x` Fragment and `autofocus` — Two Reliable Focus Mechanisms

Both were tested independently:

- `autofocus` alone: fired — browser focuses the element on DOM load.
- `#x` fragment alone: fired — browser navigates to the element with
  `id="x"`, which triggers `onfocus`.

Both work independently because they trigger focus through different browser
mechanisms. Using both together maximises cross-browser reliability — if one
mechanism behaves differently in a particular browser, the other acts as
fallback.

## Final Payload Breakdown

| Fragment | Role |
|---|---|
| `<my-cus` | Custom tag — not on WAF blocklist, passes through to browser |
| `onfocus="alert(document.cookie)"` | Event handler — executes when element receives focus |
| `id="x"` | Allows `#x` URL fragment to target this specific element |
| `tabindex="1"` | Makes the custom element focusable — required for onfocus to work |
| `autofocus` | Instructs browser to focus this element automatically on page load |
| `#x` | URL fragment — browser navigates to element with id="x", triggering onfocus |

## Browser Execution Flow

```
Victim clicks malicious link
      ↓
Browser sends GET request with crafted search parameter
      ↓
WAF checks tag name — custom tag not on blocklist, passes through
      ↓
Server reflects input into HTML response
      ↓
Browser parses custom tag as DOM element
      ↓
autofocus / #x fragment directs focus to element
      ↓
onfocus fires → alert(document.cookie) executes in victim's session
```

## Lessons Learned

- WAFs that block by tag name can be bypassed with custom HTML tags the browser treats them as valid elements and processes their attributes normally.
- Attributes are the real execution surface in HTML context, not just tags. A WAF that doesn't filter attributes independently is incomplete.
- `autofocus` and `tabindex` together make any element self-focusing no user interaction required for onfocus-based payloads.
- Two independent focus mechanisms (`autofocus` and `#x`) used together improve cross-browser reliability. 
- When a WAF blocks standard tags, enumerate what it actually checks. Test custom tags, SVG tags, MathML tags — the blocklist is rarely exhaustive.
- Always test attributes independently of tags — many WAFs treat them as separate concerns and miss attribute-based event handlers.
- In a real engagement, reflected XSS must be paired with a delivery method to demonstrate impact. A crafted URL sent via phishing or embedded in a redirect is the standard proof-of-concept format.
- Automatic execution (no user interaction) significantly increases severity rating of an XSS finding in a bug bounty report.
