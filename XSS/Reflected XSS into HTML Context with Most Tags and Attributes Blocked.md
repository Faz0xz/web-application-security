# Reflected XSS into HTML Context with Most Tags and Attributes Blocked

**Source:** PortSwigger Web Security Academy  
**Category:** Reflected XSS — WAF Bypass  
**Difficulty:** Practitioner  

## Objective

 This lab contains a reflected XSS vulnerability in the search functionality but uses a web application firewall (WAF) to protect against common XSS vectors.

To solve the lab, perform a cross-site scripting attack that bypasses the WAF and calls the print() function. 
The search functionality reflects input into HTML but a WAF blocks most standard tags and attributes. 
Goal: enumerate what the WAF allows, find an attribute that can fire automatically, deliver the exploit to a victim, and call `print()`.

## Methodology Applied

This lab required working through four sequential questions:

```
1. What tags does the WAF allow?
2. What attributes does the WAF allow?
3. Which allowed attributes can fire without user interaction?
4. How do I deliver the exploit to a victim automatically?
```

## Source

The `search` GET parameter — reflected directly into the HTML response body.

## Step 1 — Confirming the WAF and Enumerating Tags

Test input: `<img src="0" onerror=alert()>`

Result: WAF returned `"tag not allowed"` confirming a tag-based blocklist.

Rather than testing tags manually, automated enumeration using Burp Suite
Intruder:

- Intercepted the search request in Burp
- Sent to Intruder, set payload position around the tag name: `<§tag§>`
- Loaded the PortSwigger XSS cheat sheet tag list as the payload
- Ran Sniper attack filtered results by response length to find 200s

Tags that passed the WAF:
- `body`
- Custom tags (e.g. `<my-cus>`)

All standard HTML tags blocked.

## Step 2 — Enumerating Attributes

With a passing tag confirmed, tested attributes using the same approach:

- Set payload as `<my-cus §attribute§=x>` in Intruder
- Loaded the PortSwigger XSS cheat sheet event attribute list
- Ran Sniper attack large number of attributes returned 200

Allowed attributes included (among others):
`onresize`, `onbeforeinput`, `onbeforematch`, `onpointercancel`,
`onscrollend`, `onsuspend`, `onwebkitfullscreenchange`, `onafterprint`

Most standard interactive events were blocked. The allowed list consisted
largely of obscure and less-common browser events.

## Step 3 — Selecting the Right Tag and Attribute

Attempted: `<my-cus onresize=print()>`

Result: did not fire even after resizing the browser window.

Attempted: `<body onresize=print()>`

Result: fired when the browser window was resized.

**Why body works but custom tags don't:**

`onresize` is not a generic DOM event. It is tied to the browser viewport the visible area of the browser window. The `<body>` element has a direct connection to the `window` object in the browser, so `body onresize` is
effectively `window.onresize`. Resizing the browser window resizes the viewport, which fires the event on body.

A custom tag is treated by the browser as an unknown element with no special privileges. It has no connection to the window object. Resizing the viewport does not trigger anything on it. The WAF allows both — the browser is what differentiates them.

This is why tag selection matters beyond WAF bypass: the tag must both pass the WAF *and* have the native browser behaviour required for the chosen attribute to fire.

## Step 4 — Making It Fire Automatically

`<body onresize=print()>` fires on manual window resize this requires user interaction and is not a reliable exploit. The goal is automatic execution with no user action.

The solution: deliver the payload inside an `<iframe>` from an attacker-controlled page (exploit server). The iframe loads the vulnerable lab URL with the payload in the search parameter. The iframe's own `onload` attribute resizes the frame when it finishes loading — which resizes the body inside it — which triggers `onresize` automatically.

## Exploit Delivery

Payload sent from exploit server:

```html
<iframe src="LAB_URL/?search=%22%3E%3Cbody+onresize%3Dprint()%3E"
onload="this.style.width='100px'">
```

Decoded search parameter: `"<body onresize=print()>`

## Why the iframe Works — Full Chain

The iframe does two jobs simultaneously:
- Loads the vulnerable lab page with the payload injected via search parameter
- Its own `onload` fires when the page finishes loading and sets `this.style.width='100px'` — resizing the iframe

When the iframe resizes, the page loaded inside it resizes. The injected `<body onresize=print()>` fires. `print()` executes — no user interaction required.

```
Victim visits exploit server page
      ↓
iframe loads vulnerable lab URL with payload in search parameter
      ↓
WAF checks: body → allowed, onresize → allowed — passes through
      ↓
Lab page reflects "><body onresize=print()> into HTML response
      ↓
iframe finishes loading — onload fires
      ↓
iframe width changes to 100px — body inside iframe resizes
      ↓
onresize fires → print() executes automatically in victim's browser
```

## Final Payload Breakdown

| Fragment | Role |
|---|---|
| `<body` | Allowed tag — connected to window object, responds to viewport resize |
| `onresize=print()` | Allowed attribute — fires when viewport/iframe resizes |
| `onload=this.style.width='100px'` | On the iframe — triggers the resize that fires onresize |

## Lessons Learned

- WAF enumeration should be automated with Burp Intruder from the start manual testing is too slow and incomplete.
- A tag passing the WAF is necessary but not sufficient. The browser must also give that tag the native behaviour needed for the chosen event to fire.
- `onresize` only works on `body` because body is connected to the window object. Custom tags have no such connection despite passing the WAF.
- Automatic execution often requires a delivery mechanism — the exploit server simulating an attacker-controlled domain is the standard approach.
- The iframe is not just a delivery vehicle — its own attributes can be used as the trigger that makes the payload fire without user interaction.
- WAF bypass methodology: enumerate tags → enumerate attributes → identify auto-fire candidates → build delivery mechanism. This sequence applies to every WAF bypass scenario.
- Understanding browser behaviour is as important as understanding the WAF. The WAF is the server-side control. The browser is the execution environment. Both must be accounted for.
- In real engagements, automatic execution significantly increases the severity of an XSS finding no user interaction means the attack works against any victim who loads the page.
- Burp Intruder with cheat sheet payloads is the standard professional approach to WAF enumeration not manual guessing.
