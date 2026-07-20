# Reflected XSS with Some SVG Markup Allowed

**Source:** PortSwigger Web Security Academy  
**Category:** Reflected XSS — WAF Bypass  
**Difficulty:** Practitioner  

## Objective

This lab has a simple reflected XSS vulnerability. The site is blocking common tags but misses some SVG tags and events. To solve the lab, perform a cross-site scripting attack that calls the alert() function. 

The search functionality reflects user input into HTML but a WAF blocks common tags and event attributes. Goal: identify allowed SVG markup, find an event that fires automatically, and execute `alert()`.

## Source

The `search` GET parameter — reflected directly into the HTML response body.

## Step 1 — Confirming the WAF and Enumerating Tags

Initial test with standard payloads confirmed a WAF blocking common HTML tags:

Tags blocked: `<script>`, `<img>`, `<body>`, `<iframe>`

Rather than testing manually, automated tag enumeration using Burp Suite
Intruder:

- Intercepted search request, sent to Intruder
- Set payload position around tag name
- Loaded PortSwigger XSS cheat sheet tag list as payload wordlist
- Ran Sniper attack — filtered by response to find passing tags

Tags that bypassed the WAF:

- `<svg>`
- `<image>`
- `<animateTransform>`
- `<title>`
<img width="1322" height="76" alt="Screenshot 2026-07-20 214944" src="https://github.com/user-attachments/assets/f172956a-bffc-42c1-9e27-ee26a36dc463" />

The WAF was blocking standard HTML elements but had not accounted for SVG markup, which is a separate browser-supported language with its own element set.

## Step 2 — Enumerating Event Attributes

With allowed tags confirmed, the next step was finding allowed event handlers.
Manually tested common HTML event attributes — all blocked:

`onclick`, `onmouseover`, `onfocus`, `onload`, `onerror`

Switched to Burp Intruder again with the PortSwigger event attribute wordlist. Results confirmed all common HTML event attributes were blocked.

## Step 3 — Incorrect Assumption and What It Cost

At this point I formed the hypothesis: the WAF blocks all event handler attributes.

This assumption was wrong and it caused wasted time investigating SVG animation syntax rather than staying on the attack surface.

The critical observation I missed: SVG has its own event model, separate from HTML. The Intruder wordlist I used was populated with common HTML events. SVG introduces its own event attributes — including ones that common WAF rules do not account for.

A hypothesis should always be validated, not assumed. "All common HTML events are blocked" does not mean "all events in all browser technologies are blocked."

## Step 4 — SVG's Own Event Model

SVG is not just an image format. It is a browser-supported markup language with its own elements, attributes, and JavaScript event model. SVG animation elements like `<animateTransform>` have their own lifecycle events:

- `onbegin` — fires when the animation starts
- `onend` — fires when the animation ends
- `onrepeat` — fires on each repeat

These are SVG-specific. They are not HTML events and are frequently absent from standard WAF blocklists. Again I used burpsuite's intruder to find which event attributes for `<animateTransform>` are allowed by the WAF. Only one attribute `onbegin` was allowed. `onbegin` allows us to run code when it's called which is exactly what we need to trigger `alert()`. 

`onbegin` is particularly useful because SVG animations start automatically as part of the browser's normal page rendering — no user interaction required.

## Why `onbegin` Fires Automatically

```
Page loads
      ↓
Browser parses SVG markup
      ↓
<animateTransform> animation starts as part of rendering lifecycle
      ↓
onbegin fires at animation start
      ↓
alert() executes
```

The animation is not triggered by a click or hover — it starts because the browser renders SVG animations on page load. `onbegin` hooks into that automatic lifecycle, making it a zero-interaction trigger.

## Final Payload

```html
<svg><animateTransform onbegin="alert()">
```

## Payload Breakdown

| Fragment | Role |
|---|---|
| `<svg>` | Opens SVG context — allowed by WAF, not on HTML blocklist |
| `<animateTransform>` | SVG animation element — starts automatically on page render |
| `onbegin="alert()"` | SVG lifecycle event — fires when animation starts, executes alert() |

## Browser Execution Flow

```
Search parameter submitted
      ↓
WAF checks tags — svg and animateTransform not on blocklist, pass through
      ↓
WAF checks events — onbegin not on HTML event blocklist, passes through
      ↓
Server reflects input into HTML response
      ↓
Browser parses SVG markup
      ↓
animateTransform animation starts automatically during page render
      ↓
onbegin fires → alert() executes
```

## Lessons Learned

- WAF enumeration must account for the full browser attack surface, not just HTML. SVG, MathML, and other browser-supported languages each have their own elements and event models that WAF rules frequently overlook. end`, `onrepeat`) are zero-interaction triggers because animations start automatically as part of page rendering.
- Understanding a technology's attack surface is more useful than learning the technology fully. I did not need to understand SVG animations in depth
— I needed to know that SVG has its own event model and that WAF missed it.
- When a WAF blocks all standard payloads, enumerate less common browser technologies: SVG, MathML, HTML5 media elements, template tags.
- Validate every hypothesis. "The WAF blocks X" and "therefore the WAF blocks Y" are two separate claims that both require testing.
