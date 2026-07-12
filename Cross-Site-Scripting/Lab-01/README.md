### Reflected Cross-Site Scripting (XSS) into HTML Context with Nothing Encoded

**Category:** Cross-Site Scripting (XSS)                                                                                 
**Difficulty:** Apprentice                                                                                
**Platform:** PortSwigger Web Security Academy

### Overview

This lab demonstrates the most basic form of a reflected XSS vulnerability: a search feature that takes user input and reflects it
straight back into the page's HTML with no encoding or sanitization whatsoever. The objective was to perform a cross-site scripting
attack that calls the alert function.


### Vulnerability

The site is a blog with a search box. Whatever gets typed into that box is echoed back into the resulting page, likely
rendered server-side similar to:
```
html<p>0 search results for '<search term>'</p>
```
Because the search term is inserted directly into the HTML response without any HTML-encoding (no escaping of <, >, ", etc.),
any HTML or JavaScript submitted through the search box gets interpreted by the browser exactly as written, rather than being
displayed as plain text.

### Methodology

1. **Locate the Injection Point**
   The search box on the blog's homepage was the obvious target — search functionality almost always reflects the query
   term back into the results page.
   
2. **Test a Basic Script Payload**
   The following payload was entered directly into the search field:
   ```
   html   <script>alert(1)</script>
   ```
   
   
3. **Submit and Observe**
   Clicking "Search" sent the payload to the server as a query string parameter:
   Because the response reflected this value with zero encoding, the browser parsed ```<script>alert(1)</script>``` as a real 
   script tag and executed it immediately, popping a JavaScript alert box:
   

4. **Confirm the Lab is Solved**
   The alert firing on the page is enough to satisfy the lab's success condition, flipping the lab status from
   "Not solved" to "Solved."

### Why This Works

1. The application takes the search parameter from the URL and inserts it directly into the HTML response.
2. No output encoding is applied — special characters like < and > are passed through unchanged instead of being converted
   to &lt; and &gt;.
3. Because the raw <script> tag reaches the browser untouched, the browser has no way to distinguish it from a legitimate
   script placed by the developer, so it executes it.
4. Since the payload comes from the URL (a GET parameter) and is only reflected in the immediate response, this is a
   reflected XSS — it requires the victim to click a malicious link or submit a crafted request, rather than being
   permanently stored on the server.

### Impact

An attacker can craft a malicious URL containing a script payload and trick a victim into clicking it
(e.g., via phishing email or a link on another site). If the victim is logged in, that script runs in the context of their
authenticated session and can be used to:
1. Steal session cookies or tokens, potentially leading to full account takeover.
2. Perform actions on the victim's behalf without their knowledge (e.g., changing account details, making purchases).
3. Redirect the victim to a phishing page or serve malware.
4. Deface the page or inject fake content to deceive the user.

### Remediation

1. HTML-encode all user-controllable output before rendering it into the page — convert <, >, ", ', and & into their
   corresponding HTML entities.
2. Use context-aware output encoding, since encoding requirements differ between HTML body, attributes, JavaScript, and
   URL contexts.
3. Apply a Content Security Policy (CSP) to restrict which scripts are allowed to execute, reducing the impact even if a
   payload slips through.
4. Use templating engines/frameworks that auto-escape output by default (e.g., React, Django templates, Jinja2) rather than
   manually concatenating strings into HTML.
5. Treat all user input as untrusted, regardless of source (URL parameters, form fields, headers, cookies).

### Payload Summary
```
html<script>alert(1)</script>
```
