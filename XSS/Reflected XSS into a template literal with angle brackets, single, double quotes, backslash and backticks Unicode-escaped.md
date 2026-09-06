# Reflected XSS into a template literal with angle brackets, single, double quotes, backslash and backticks Unicode-escaped

**Platform:** PortSwigger Web Security Academy  
**Category:** Cross-Site Scripting (XSS)  
**Difficulty:** Apprentice  

## Objective

This lab contains a reflected cross-site scripting vulnerability in the search blog functionality. The reflection occurs inside a template string with angle brackets, single, and double quotes HTML encoded, and backticks escaped. To solve this lab, perform a cross-site scripting attack that calls the alert function inside the template string. 

## Initial Reconnaissance

I started by submitting a simple test input "L0mb@r" to see where it gets reflected within the web page. It is reflected within a  Javascript block in the web page.


``<script>
var message = `0 search results for 'L0mb@r'`;
document.getElementById('searchMessage').innerText = message;
</script>``

This showed that my input was being directly injected into a JavaScript template literal.

## Understanding the Context

The important part of the JavaScript was:

    var message = `0 search results for 'Test'`;

The backticks indicate that this is a JavaScript template literal.

Unlike normal JavaScript strings, template literals support expression interpolation using:

    ${...}

For example:

    var message = `L0mb@r ${7 * 7}`;

JavaScript evaluates the expression inside `${...}`, resulting in:

    L0mb@r 49

This meant I could potentially execute arbitrary JavaScript **without needing to break out of any surrounding quotes** I just needed to get an unescaped `${...}` sequence into the template literal.

## Testing the Template Literal

I first tested whether expressions inside `${...}` were actually being evaluated.

Payload:

<img width="518" height="88" alt="image" src="https://github.com/user-attachments/assets/33137832-d89e-40f0-b607-3fed6535051f" />


    ${7*7}

If the application evaluates the expression, the resulting message should contain:

    0 search results for 'L0mb@r 49'
    
  <img width="1023" height="390" alt="image" src="https://github.com/user-attachments/assets/17c7da2d-821f-4b91-9c14-2fe6d243c6b2" />


This confirmed that I could execute JavaScript expressions from inside the template literal.

Since `${...}` allows JavaScript expressions to be evaluated, I used the following payload:

    ${alert(1)}

The application processed it as part of the template literal:

    var message = `0 search results for '${alert(1)}'`;

The `alert(1)` expression is evaluated while the template literal is being constructed, causing the alert to execute.

## Remediation

The primary remediation is to avoid inserting untrusted user input directly into JavaScript code. Instead of constructing JavaScript using string interpolation, the application should keep the data separate from executable code.


