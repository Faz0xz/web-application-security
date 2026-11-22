# Reflected XSS with event handlers and 'href' attributes blocked

**Platform:** PortSwigger Web Security Academy  
**Vulnerability:** Reflected Cross-Site Scripting (XSS)  
**Difficulty:** Apprentice

## 1. Lab Description

This lab contains a reflected XSS vulnerability where user input is reflected into the page. The application implements an HTML tag and attribute filter. Some HTML and SVG tags are permitted, but event handler attributes and anchor `href` attributes are blocked.

**Objective:** Inject a vector that executes JavaScript's `alert()` function when clicked. The vector must contain the word **Click** so the simulated victim recognizes and clicks it.

## 2. Initial Reconnaissance

I began by examining the application's input reflection and identifying which HTML tags and attributes were accepted by the filter. Rather than guessing individual tags, I used Burp Suite Intruder to test different HTML tags and identify the ones that were permitted.

<img width="1917" height="1000" alt="Screenshot 2026-11-22 170825" src="https://github.com/user-attachments/assets/e1399f3b-e5ae-4381-86fe-e61a99494cfd" />


<img width="1875" height="943" alt="Screenshot 2026-11-22 170811" src="https://github.com/user-attachments/assets/acaef473-7ffc-480a-9e20-ed52d332dedb" />

Among the allowed tags, I identified:

- `<svg>`
- `<a>`
- `<animate>`
- `<image>`
- `<title>`

This was an important discovery because SVG provides functionality that can modify element attributes through animation.

## 3. Understanding the Vulnerability

The application blocked conventional XSS techniques involving event handlers such as `onclick` and anchor `href` attributes. A conventional payload would look like:

```html
<a href="javascript:alert(1)">Click me</a>
```

However, this approach was blocked by the application's filtering mechanism. I explored an alternative using SVG's built-in animation functionality.

### SVG `<animate>` element

The SVG `<animate>` element allows an SVG element's attributes to change over time.

For example:

```html
<circle cx="50" cy="50" r="20">
    <animate
        attributeName="r"
        from="20"
        to="40"
        dur="2s" />
</circle>
```

In this example, the circle's radius changes from 20 to 40 over two seconds. The important attribute is `attributeName`, which specifies the attribute being animated. This functionality can also be used to modify an anchor's `href` attribute dynamically.

### SVG anchor element

An SVG `<a>` element represents a hyperlink. It can contain other SVG elements, including `<text>`. When an `<animate>` element is nested inside the anchor, it can target the anchor's attributes. This allows the `href` value to be assigned through SVG animation rather than being supplied as a conventional `href` attribute in the original input.

## 4. Exploitation

The following payload was used:

```html
<svg>
    <a>
        <animate attributeName=href
                 values=javascript:alert(1) />
        <text x=20 y=20>Click me</text>
    </a>
</svg>
```

### Payload Breakdown

**`<svg>`**

Creates an SVG container, allowing SVG-specific elements and functionality to be used.

**`<a>`**

Creates an SVG hyperlink element. This is the target whose `href` attribute will be modified.

**`<animate attributeName=href>`**

Instructs the browser's SVG animation engine to modify the anchor's `href` attribute.

**`values=javascript:alert(1)`**

Specifies the value assigned to the animated attribute. When the anchor is clicked, the JavaScript URL executes `alert(1)`.

**`<text x=20 y=20>Click me</text>`**

Displays clickable text inside the anchor. The word "Click" is included to satisfy the lab's simulated victim interaction requirement.

## 5. Why the Bypass Works

The application's filtering mechanism blocks direct use of the `href` attribute. However, SVG provides an alternative mechanism for modifying an element's attributes through animation.

Instead of directly supplying:

```html
<a href="javascript:alert(1)">
```

The payload uses:

```html
<animate attributeName=href
         values=javascript:alert(1) />
```

The browser processes the SVG animation and dynamically assigns the anchor's destination. This demonstrates how restrictive HTML filters can be bypassed when they fail to account for alternative browser functionality.

<img width="1917" height="523" alt="Screenshot 2026-11-22 170743" src="https://github.com/user-attachments/assets/0f376afb-7d98-431d-9df9-c2f6abed73ac" />


## 7. Remediation

- Use a maintained HTML sanitization library with a strict allowlist.
- Remove dangerous URL schemes such as `javascript:` from URL-bearing attributes.
- Restrict SVG elements and animation features when they are not required.
- Encode untrusted input according to its output context.
- Avoid relying exclusively on custom tag and attribute blocklists.
- Deploy Content Security Policy as an additional security layer, not as a replacement for proper sanitization.
