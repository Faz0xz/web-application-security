# Cross-Site Scripting (XSS)

Cross-Site Scripting (XSS) is one of the most common web application vulnerabilities. While payloads often receive the most attention, this directory focuses on **understanding why an exploit works** rather than memorizing exploit strings.

The write-ups in this folder document my investigation process while solving XSS challenges from platforms such as PortSwigger Web Security Academy and other intentionally vulnerable applications.

---

# Learning Goals

The objective of these notes is to develop the ability to:

- Trace attacker-controlled input through an application
- Identify executable contexts
- Understand browser parsing behavior
- Build exploits based on context rather than memorization
- Develop a repeatable methodology for testing web applications

---

# XSS Investigation Methodology

After performing various labs I came up with this methodology which I apply to each lab to find and exploit an XSS vulnerability. 

```text
 Input  Source
      │
      ▼
Identify the Source
      │
      ▼
Find Every Reflection
      │
      ▼
Identify the Sink
      │
      ▼
Determine the Parser
      │
      ▼
Identify the Current Context
      │
      ▼
Understand Browser Expectations
      │
      ▼
Construct a Valid Payload
      │
      ▼
Verify Execution
      │
      ▼
Explain Why It Worked
```

The payload is intentionally the final step.

Understanding the browser's behavior is more valuable than memorizing payloads.

---

# Questions I Ask During Every Lab

Before attempting exploitation, I answer the following questions:

### Source

- Where does attacker-controlled input originate?
- Is the input provided through a URL, form field, cookie, or HTTP header?

---

### Reflection

- Is my input reflected?
- How many times?
- Which reflection appears most interesting?

---

### Sink

Where does attacker-controlled data end up?

Examples include:

- HTML body
- HTML attribute
- JavaScript
- DOM sinks
- URL attributes

---

### Parser

Which parser is interpreting my input?

Examples:

- HTML Parser
- JavaScript Parser
- URL Parser
- CSS Parser
- jQuery Selector Parser

---

### Context

Exactly where is my input located?

Examples:

```html
<p>INPUT</p>
```

```html
<input value="INPUT">
```

```javascript
var search = 'INPUT';
```

```html
<a href="INPUT">
```

The current context determines what syntax the browser expects.

---

### Trigger

How is JavaScript eventually executed?

Examples include:

- Immediately
- User click
- Image loading failure
- Mouse events
- Focus events
- Hash changes

---

# Core Principles Learned

## Reflection does not equal vulnerability

Finding your input in the response does not automatically indicate an exploitable XSS vulnerability.

The surrounding context determines whether execution is possible.

---

## Context determines payload

There is no universal XSS payload.

The payload must match the parser currently interpreting the input.

For example:

- HTML contexts require valid HTML.
- JavaScript contexts require valid JavaScript.
- URL contexts require valid URLs.

---

## Browser behavior matters more than payloads

Most successful exploits are simply valid syntax interpreted by the browser in an unexpected context.

Understanding browser behavior makes unfamiliar XSS scenarios much easier to solve.

---

## Think like the parser

Instead of asking:

> "Why didn't my payload work?"

I now ask:

> "What is the browser expecting to parse at this point?"

This mindset has been the single biggest improvement in my approach to XSS.

---

# Write-up Structure

Every write-up follows roughly the same format:

- Objective
- Reconnaissance
- Source Identification
- Reflection Analysis
- Sink Identification
- Context Analysis
- Investigation
- Root Cause
- Browser Execution Flow
- Lessons Learned

---

# Current Progress

## Reflected XSS

- [x] HTML Context
- [x] HTML Attribute Context
- [x] JavaScript String Context

## Stored XSS

- [x] HTML Context
- [x] URL (`href`) Context

## DOM XSS

- [x] `document.write()`
- [x] `innerHTML`
- [x] `href`
- [x] jQuery Selector
- [ ] More advanced DOM sinks

---

# Key Takeaways

The biggest lesson I've learned so far is that successful XSS exploitation is rarely about finding the right payload.

Instead, it is about understanding:

- where attacker-controlled input originates,
- how it flows through the application,
- which parser interprets it,
- what syntax that parser expects,
- and why the browser eventually executes JavaScript.

## 📚 Recommended Resources

### PortSwigger XSS Labs

- **z3nsh3ll** – Detailed explanations of PortSwigger labs, browser behavior, and the underlying vulnerability mechanics.
  - **YouTube:** [@z3nsh3ll](https://www.youtube.com/@z3nsh3ll)
