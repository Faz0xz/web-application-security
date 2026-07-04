# Reflected DOM XSS

**Source:** PortSwigger Web Security Academy  
**Category:** Reflected DOM XSS  
**Difficulty:** Practitioner  

## Objective

This lab demonstrates a reflected DOM vulnerability. Reflected DOM vulnerabilities occur when the server-side application processes data from a request and echoes the data in the response. A script on the page then processes the reflected data in an unsafe way, ultimately writing it to a dangerous sink. To solve this lab, create an injection that calls the alert() function. 

## Source

The `search` GET parameter — user input submitted via the search form.

<img width="1002" height="262" alt="Screenshot 2026-06-30 193536" src="https://github.com/user-attachments/assets/1006649e-a680-429a-928a-9a4d63bc8952" />


## Initial Reconnaissance

Test input: `Test")>}`

Result: input was reflected back inside an `<h1>` tag in the page body. Angle
brackets and quotes appeared to be treated as plain text in that context — not
immediately exploitable there. However, this confirmed the input was being
processed somewhere in the application, so the next step was finding *where*
it was going beyond the HTML reflection.

<img width="953" height="357" alt="Screenshot 2026-07-03 172802" src="https://github.com/user-attachments/assets/73ef070f-ab3d-4c93-86d8-4ffee0a56c4b" />


## Finding the Real Sink

Inspecting the page source revealed a `<script>` tag loading an external
JavaScript file referencing a `search-results` function. Opening that file in
developer tools revealed the critical code:

<img width="455" height="93" alt="Screenshot 2026-07-03 172943" src="https://github.com/user-attachments/assets/16c70eea-fdf5-42f8-b8e0-3b28d7fde440" />


```javascript
function search(path) {
    var xhr = new XMLHttpRequest();
    xhr.onreadystatechange = function() {
        if (this.readyState == 4 && this.status == 200) {
            eval('var searchResultsObj = ' + this.responseText);
            displaySearchResults(searchResultsObj);
        }
    };
    xhr.open("GET", path + window.location.search);
    xhr.send();
}
```

`eval()` takes a string and executes it as JavaScript — whatever the server
returns gets fed directly into it. This is the sink. The server's response
would be concatenated into the eval string, meaning any input that survived
into that response could be executed as code.

## Data Flow

```
search parameter
      ↓
GET request to server
      ↓
server returns JSON response containing search term
      ↓
client-side JS concatenates JSON into eval() string
      ↓
eval() executes the full string as JavaScript
      ↓
displaySearchResults() renders output into DOM
```

## Inspecting the Server Response

Opening Burp Suite and checking the HTTP response for the search request
revealed the server returns JSON, this can also be done via developer tools in the browsers by sending the request and looking for it in the network tab:

```json
{"results":[],"searchTerm":"a' <img src=\"0\" onerror=alert()"}
```

<img width="572" height="97" alt="Screenshot 2026-07-03 173006" src="https://github.com/user-attachments/assets/14e2679e-3bf3-492f-bffd-a17f501ad6f5" />


Two things became clear:

1. My input lands inside the `searchTerm` value — wrapped in a JSON string,
   which uses `\"` to escape double quotes within a string value rather than
   terminating it.
2. That JSON gets concatenated directly into the `eval()` call, so the full
   string eval sees is:
   `var searchResultsObj = {"results":[],"searchTerm":"INPUT"}`

The injection point is the `searchTerm` value inside that JSON string.

## Parser Context

Two sequential contexts to escape — both must be handled in the right order:

**Context 1: JSON string**  
Inside a JSON string, `\"` is an escaped quote (stays inside the string).
A `\\` is an escaped backslash — produces one literal `\` character.

**Context 2: JavaScript (via eval)**  
Once the JSON string is broken, the remainder executes as raw JavaScript
inside `eval()`. Standard JS comment `//` discards everything after it on
the same line.

## Failed Attempts

| Input | Why it failed |
|---|---|
| `a"alert()//` | The `"` alone gets JSON-encoded to `\"` — still inside the JSON string, never escapes |
| `a//"alert()` | The `//` is inside the JSON string — it's just text at that point, not a JS comment |

The key realisation: I needed to escape two contexts in sequence — JSON first,
then JavaScript — not one or the other.

## Payload Construction

```
a\"-alert()}//
```

**What happens at the JSON layer:**

My `\` gets JSON-encoded to `\\`. My `"` gets JSON-encoded to `\"`.

The JSON response becomes:
```json
{"results":[],"searchTerm":"a\\"-alert()}//"}
```

The JSON parser reads `\\` as one literal backslash character. That backslash
is now just data — it has no escaping power left. The `"` that follows it
therefore terminates the JSON string.

**What happens at the JavaScript layer:**

After escaping the JSON string, `eval()` is processing the full object literal:

```javascript
var searchResultsObj = {"results":[],"searchTerm":"a\"-alert()}//"}
```

Three things still need to be handled:

**The `}` — closing the object literal**

An object literal is any value written directly as `{key: value}` in code.
The entire server response `{"results":[],"searchTerm":"..."}` is an object
literal being assigned to `searchResultsObj`. When the `"` in the payload
terminates the `searchTerm` string, the `{` that opened the object is still
unclosed. JavaScript won't execute anything inside an unclosed structure —
the engine is still waiting for `}` to complete the assignment.

Adding `}` closes the object. The `var` assignment is now complete and valid.

**The `-` — forcing expression mode**

After the object closes, the engine hits `alert()` as the next token. Two
values sitting directly next to each other with no operator between them is
ambiguous — the engine can misread `alert()` as an attempt to invoke the
preceding object as a function. Objects aren't callable, so this throws a
TypeError before alert ever runs.

The `-` places an explicit operator between them:

```javascript
{...} - alert()
```

Now the engine evaluates both sides as operands of subtraction. `alert()`
gets evaluated as a value — which means it executes. The result is `NaN`
(subtracting `undefined` from an object). The result doesn't matter.
`alert()` already fired.

**The `//` — discarding the remainder**

The `//` comments out the trailing `"}` left over from the original JSON
structure, preventing a syntax error from the dangling characters.

## Final Payload Breakdown

| Fragment | Layer | Role |
|---|---|---|
| `\` | JSON | Forces JSON to produce a literal `\` in output, consuming its own escaping power |
| `"` | JSON → JS | Terminates the JSON string once the `\` is neutralised |
| `}` | JavaScript | Closes the unclosed object literal — completes the var assignment |
| `-` | JavaScript | Places an operator between the closed object and alert(), forcing the engine into expression evaluation mode so alert() actually runs |
| `alert()` | JavaScript | Executes in global scope via eval() |
| `//` | JavaScript | Comments out trailing `"}` — prevents syntax error in eval() |

## Browser Execution Flow

```
Search input submitted
      ↓
GET request sent — Burp confirms JSON response structure
      ↓
Client-side JS concatenates JSON into eval() string
      ↓
JSON parser: \\ → literal backslash, " → terminates string
      ↓
} closes the object literal — var assignment completes
      ↓
- forces expression mode — alert() evaluated as operand
      ↓
// discards remaining "}
      ↓
alert() executes in global scope
```

## Lessons Learned

- The HTML reflection of input is not always the exploitable sink — always
  check how the application processes input beyond what's visible on the page.
- Check server responses in Burp or DevTools Network tab. If the server
  returns JSON containing your input, that JSON is likely being consumed
  by client-side JavaScript — trace where it goes.
- `eval()` is an immediately recognisable dangerous sink. Any input reaching
  it that isn't fully sanitised is potentially exploitable.
- Escaping two sequential contexts requires solving them in order — JSON
  first, JavaScript second. Solving them in the wrong order or conflating
  them is why early attempts failed.
- A backslash in the payload isn't for JavaScript — it's for JSON. A `//`
  comment isn't for JSON — it's for JavaScript. Each character serves a
  specific parser at a specific layer.

## Professional Takeaways

- Always inspect external JS files loaded by the page — sink logic is
  frequently in client-side scripts, not inline HTML.
- JSON responses containing user input are a strong signal of potential
  DOM XSS — identify what client-side code consumes them.
- When exploitation fails, ask which context you're currently in and
  whether you've fully escaped it before moving to the next layer.
