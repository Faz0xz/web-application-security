# DOM XSS in AngularJS Expression with Angle Brackets and Double Quotes HTML-Encoded

**Source:** PortSwigger Web Security Academy  
**Category:** DOM XSS  
**Difficulty:** Practitioner  

## Objective

The search functionality reflects input inside an AngularJS template expression. Angle brackets and double quotes are HTML-encoded. Goal: execute `alert()` by exploiting AngularJS expression evaluation.

## Initial Reconnaissance

- Input source: `search` GET parameter.
- The page uses AngularJS — identifiable by the `ng-app` attribute on an HTML element.
- AngularJS scans nodes marked with `ng-app` and evaluates anything inside `{{ }}` as a JavaScript expression.

## Encoding Test

Input: `TEST>}"`

Observed behavior:
- `>` was HTML-encoded → angle brackets are blocked, HTML injection not viable.
- `"` was HTML-encoded → double quotes blocked.
- `}` was **not** encoded → curly braces are available.

Conclusion: the injection path is through AngularJS template expressions using `{{ }}`, not raw HTML or attribute injection.

## Confirming Expression Evaluation

Input: `{{2+2}}`

Result: page rendered `4`, not the literal string `{{2+2}}`.

Conclusion: AngularJS is active and evaluating expressions inside the search reflection. The sink is the AngularJS template engine itself.

## Data Flow

```
search parameter
      ↓
server reflects value into HTML node marked with ng-app
      ↓
AngularJS scans node, finds {{ }} expression
      ↓
AngularJS evaluates expression in its sandboxed scope
      ↓
result rendered into DOM
```

## Sink

AngularJS template expression evaluation — any `{{ }}` content inside an `ng-app` node is treated as executable JavaScript by the framework.

## First Exploitation Attempt

Input: `{{alert()}}`

Result: did not fire.

Reason: AngularJS evaluates expressions inside its own sandboxed scope, which deliberately blocks access to browser globals like `window` and `alert`. Inside the sandbox, `alert` simply does not exist — it is a property of `window`, which the sandbox cannot see. Calling `alert()` directly is blocked not because `alert` is broken, but because the sandbox prevents any path to `window`.

## Understanding the Sandbox Escape

The sandbox blocks direct references to globals. But it does permit access to properties of legitimate AngularJS scope objects.

Key insight: `$on` is a built-in AngularJS scope method — and it is a function. In JavaScript, every function carries a `.constructor` property that points back to the built-in `Function` constructor — the thing responsible for creating all functions.

Verified in browser console:
```javascript
function test() {}
console.log(test.constructor) // → Function
```

`Function` accepts a string and creates a new function whose body is that string:
```javascript
Function('alert(1)') // equivalent to: function() { alert(1) }
```

Crucially, functions created this way execute in **global scope**, not inside the AngularJS sandbox — so they can see `window.alert`.

The sandbox never blocked `.constructor` because `$on` is a legitimate scope object. The sandbox permitted the property access, but that property led directly to `Function`, which operates outside the sandbox entirely.

## Final Payload

```
{{$on.constructor('alert(1)')()}}
```

## Payload Breakdown

| Fragment | Role |
|---|---|
| `{{` | Opens AngularJS template expression |
| `$on` | Legitimate AngularJS scope function — permitted by sandbox |
| `.constructor` | Accesses the built-in `Function` constructor via `$on` |
| `('alert(1)')` | Passes `alert(1)` as a string — creates a new function with that body |
| `()` | Immediately invokes the newly created function |
| `}}` | Closes AngularJS expression |

## Why This Bypasses the Sandbox

The sandbox was never broken — the payload stayed inside it the entire time. The bypass worked because:

1. The sandbox permits access to properties of scope objects like `$on`.
2. `.constructor` on any function returns `Function` — a global built-in the sandbox forgot to restrict.
3. `Function('alert(1)')` builds and executes code in global scope, where `window` and `alert` are fully accessible.

The technique: use what the sandbox *allows* to reach what the sandbox *blocks*.

## Browser Execution Flow

```
HTTP Response
      ↓
HTML Parser builds DOM — encodes < > and "
      ↓
AngularJS finds ng-app node
      ↓
AngularJS evaluates {{ }} expression in sandbox
      ↓
$on.constructor resolves to Function
      ↓
Function('alert(1)') creates new function in global scope
      ↓
() invokes it → alert() fires
```

## Lessons Learned

- HTML encoding defends the HTML parser context. It does nothing for AngularJS expression evaluation — a completely separate execution path.
- Framework-level template engines introduce their own attack surface independent of standard XSS vectors.
- Sandboxes define what is explicitly permitted, not what is explicitly blocked. The gap between those two things is where bypasses live.
- `.constructor` on any function reaches `Function` — a fundamental JavaScript property that sandbox implementations frequently overlook.
- Confirming the execution environment (`{{2+2}}`) before attempting exploitation is essential methodology — the payload depends entirely on knowing what is evaluating your input.

## Professional Takeaways

- Always identify what framework is running on a page before testing. `ng-app`, `ng-controller`, `v-bind`, `data-react` — these all signal non-standard execution contexts with their own injection surfaces.
- Expression injection in template engines is a distinct vulnerability class from reflected/stored XSS. The parser is the framework, not the browser's HTML engine.
- When direct payloads are blocked by a sandbox, look for legitimate objects inside the sandbox whose properties provide a path to global scope.
