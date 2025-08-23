---
icon: spider
layout:
  width: default
  title:
    visible: true
  description:
    visible: false
  tableOfContents:
    visible: true
  outline:
    visible: true
  pagination:
    visible: true
  metadata:
    visible: true
---

# Bug Bounty Initial Steps.

### 1. Knowing the App and Reconnaissance

One of the most common procedures is to start by reviewing a web application's front-end components, such as HTML, CSS, and JavaScript (also known as the "front-end trinity"), and attempt to find vulnerabilities like Sensitive Data Exposure and Cross-Site Scripting (XSS).

* **Use the app like a regular user.** Discover its purpose, how to use it, and find hidden or less popular settings and functions.
* **Start inspecting the requests.** Try to find requests that contain direct references to database objects.
* Remember to **highlight** important requests in Burp Suite's Proxy history.
* Remember to **rename** your Burp Suite Repeater tabs to stay organized.

***

### 2. Objectives / What to Look For

#### Insecure Direct Object Reference (IDOR)

If you see a parameter like `userId=100`, try changing it to `userId=101` to see if you can access another user's data.

*   IDOR with direct reference to static files:

    example.com/static/123.txt --> try accessing example.com/static/456.txt
*   IDOR with direct reference to a database object:

    example.com/customer?account=123 --> try accessing example.com/customer?account=456
*   IDOR in the URL path:

    example.com/app/task/347/delete --> try changing the number to 348.
* **IDOR with non-numeric IDs:** Instead of an incrementing number, an application might use Globally Unique Identifiers (GUIDs) to identify users. This may prevent an attacker from guessing another user's identifier. However, the GUIDs belonging to other users might still be disclosed elsewhere in the application, such as in user messages or reviews, allowing for IDOR.

#### Expanding on IDOR Techniques

Your current IDOR section is great for basic numeric IDs. Here are some common variations you should also be looking for.

*   Test different HTTP Methods on the same endpoint:

    If you find an endpoint like GET /api/v1/users/123 that displays user data, don't stop there. Try other methods on that same endpoint. The developers might have forgotten to apply proper access control to other verbs.

    * `PUT /api/v1/users/123` (Could let you modify another user's data)
    * `POST /api/v1/users/123` (Could let you add a sub-object, like a comment, as another user)
    * `DELETE /api/v1/users/123` (Could let you delete another user)
*   Look for IDORs in Request Bodies and Headers:

    Identifiers aren't always in the URL. Modern applications frequently send identifiers in the body of POST or PUT requests, often in JSON format. Always check these for potential IDORs.

    JSON

    ```
    POST /api/v1/profile/update HTTP/1.1

    {
      "userId": "12345",
      "bio": "This is my new bio."
    }

    ```

    Change `userId` to someone else's ID (`"12346"`) and see if you can update their bio.
*   Test for Hashed or Obfuscated IDs:

    Sometimes you'll see IDs that aren't simple numbers, like c29tZXVzZXJpZA== (Base64) or a5d8a245-2f3c-4b68-8a13-3a5e1e7f9a1b (UUID).

    1. **Decode them.** Use a tool like Burp's Decoder to see if they are just encoded versions of a number (e.g., `c29tZXVzZXJpZA==` decodes to `someuserid`).
    2. **Find where they leak.** Even if you can't guess a UUID, the application might leak other users' UUIDs in different places (e.g., in a public comments section, a list of group members, etc.). Grab another user's leaked UUID and use it in requests.

#### Expanding on Access Control & Parameter Tampering

These techniques focus on escalating privileges vertically (user to admin) or bypassing restrictions.

*   Force Browsing to Privileged Endpoints:

    Just because you don't see a link to an admin panel in the UI doesn't mean it doesn't exist. Always try to manually browse to common administrative paths after logging in as a regular user.

    * `/admin`
    * `/dashboard`
    * `/console`
    * `/admin-panel`
    * `/manage`
*   Testing Multi-step Process Vulnerabilities:

    Applications often have workflows where you must complete Step A before Step B. A common access control flaw is failing to verify that you've completed the previous steps.

    * **Example:** In an e-commerce site, try to access the `/checkout` page with items in your cart. Then, empty your cart and try to access `/checkout` again directly. Or, try to call the API endpoint for "complete purchase" without ever adding an item to the cart. Can you cause unexpected behavior or access a function you shouldn't be able to?

#### Account Takeover

```
![GitLab Bug](https://pbs.twimg.com/media/Gkt1eMjWUAAibig?format=jpg)
The security researcher used a technique known as **HTTP parameter pollution** combined with a content-type conversion.
```

* **Intercept the Request:** The hunter used a web proxy tool called **Burp Suite** to intercept the "Forgot Password" request sent to GitLab.
* **Modify the Request:** They converted the request's body to JSON format.
* **Pollute the Parameter:** They changed the `user[email]` parameter from a single email string (e.g., `"victim@gmail.com"`) to a JSON array containing two emails: the victim's and the attacker's (e.g., `["victim@gmail.com", "attacker@gmail.com"]`).
* **Exploit the Vulnerability:** GitLab's backend processed this array by sending the password reset link to _both_ email addresses.

***

Very common and critical finding. An attacker finds an API endpoint or function that modifies user data, but it fails to properly check if the logged-in user is authorized to perform the action on the target account.

* **How it's found:** A bug hunter logs into their own account and intercepts a request to change their email, for example: `POST /api/v2/user/change-email` `{"user_id": "12345", "new_email": "hunter@example.com"}`
* **The Exploit:** The hunter then replays the request but changes the `"user_id"` to that of a victim (e.g., `"54321"`). If the server doesn't validate that the user making the request is user `12345`, it will blindly change the email for user `54321`.
* **Result:** The attacker changes the victim's email address to one they control, then uses the "Forgot Password" flow to take over the account completely.

#### Access Control

Log in as User A and send a request that accesses a private function to Repeater. Then, log out and log in as User B. In Repeater, replace the session cookie from the original request with User B's cookie and resend it. Can User B access User A's function?

#### Parameter Tampering

Change values like prices, quantities, or user roles in a request to see if the server validates them properly.

*   Boolean or role-based parameters:

    Look for parameters in requests like admin=false or role=1 and try changing them to admin=true or role=2.
*   Mass assignment / Parameter addition:

    In a request like {"email":"test@mail.com"}, try adding a new parameter to escalate privileges:

    {"email":"test@mail.com", "roleid": 2}
*   HTTP Verb Tampering:

    Replace the POST method with GET. For example, change:

    POST /admin-roles HTTP/2

    to:

    GET /admin-roles?username=user\&action=upgrade HTTP/2

    (Tip: In Burp Suite, you can do this easily by right-clicking the request and selecting "Change request method".)
*   Bypassing controls with headers:

    Use headers like X-Original-URL or X-Rewrite-URL to try to access admin functionality.

    HTTP

    ```
    POST / HTTP/1.1
    Host: example.com
    X-Original-URL: /admin/deleteUser

    ```

#### Try to:

```
- Modify/Delete users  
- Modify/Delete carts  
- Access personal information / files  
- Do actions in their name: comment, fav, post, etc  
```

#### JavaScript Analysis

*   Leaked paths:

    Search JavaScript files for hardcoded URLs, which may reveal sensitive endpoints.

    if(admin) --> ('href', 'https://insecure-website.com/administrator-panel-yb556');
*   Leaked secrets:

    Scan JavaScript files for sensitive information like passwords, API keys, secrets, or usernames.

#### Personal Information Leaks

* Look for endpoints that leak Personally Identifiable Information (PII) like names, addresses, or email addresses, especially if they are not meant to be public.
* Check for sensitive EXIF data in photos, which can expose metadata like GPS locations.

***

### 3. Useful Burp Suite Tools

* **HTTP Smuggler:** An extension for finding HTTP Request Smuggling vulnerabilities.
* **Autorize:** An extension for automating authorization testing.
* **JS Link Finder:** An extension for finding links and endpoints within JS files.
