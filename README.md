#Web-Application Security

Write-ups documenting vulnerability analysis and exploitation reasoning from hands-on labs and self-directed learning.

This is not a collection of payloads or copy-paste solutions. Each entry walks through the actual investigation: identifying the source, tracing data flow, locating the sink and parser context, 
testing hypotheses (including the ones that failed), and explaining why the final exploit works.

Organized by vulnerability class, not by platform. Each lab notes its source (PortSwigger, TryHackMe, etc.) 
but the folder structure reflects the vulnerability type, since the same class of bug shows up across multiple platforms over time.

xss/           Cross-site scripting — reflected, stored, DOM-based

sqli/          SQL injection

ssrf/          Server-side request forgery

fundamentals/  Browser parsing, DOM, source/sink theory


Each write-up focuses on understanding browser behavior, parser contexts, data flow, and exploitation techniques rather than memorizing payloads.

As I progress through the academy, this repository will grow into a collection of notes covering 
web application penetration testing concepts and practical exploitation techniques.
