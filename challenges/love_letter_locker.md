# Love Letter Locker

**Category:** Web Exploitation  
**Vulnerability:** Insecure Direct Object Reference (IDOR)  
**Difficulty:** Easy

---

## TL;DR

A web app stores user-created letters accessible via sequential integer IDs in the URL. No authorisation check is performed on letter access, allowing any authenticated user to read any other user's letters by simply changing the ID in the URL. The flag is in letter `/letter/1`.

---

## Reconnaissance

The challenge is tagged as Web, so no port scanning is needed — we focus entirely on the web application.

![IMG 1](/challenges/screenshots/love_letter_locker/image.png)


## Identifying the Attack Vector

The app allows users to register, log in, and write letters. After creating an account and writing a letter, we get redirected to `/letters`. 

![IMG 2](/challenges/screenshots/love_letter_locker/image(1).png)

We can then open our letter and get redirected to :

```
/letter/3
```

![IMG 3](/challenges/screenshots/love_letter_locker/image(2).png)


The sequential integer ID is the red flag here. There is no token, no hash, no user binding in the URL — just a number. This is a textbook IDOR setup.

A hint in the UI ("Tip from Cupid") also nudges toward this vulnerability.

---

## Exploitation

### Step 1 — Observe Your Own Letter URL

After writing a letter, note the URL:

```
http://<TARGET_IP>/letter/3
```

### Step 2 — Enumerate Other Letters

Manually change the ID to access other users' letters:

```
http://<TARGET_IP>/letter/2  → accessible
http://<TARGET_IP>/letter/1  → flag
```

![IMG 4](/challenges/screenshots/love_letter_locker/image(4).png)

---

## Remediation

**1. Enforce object-level authorisation checks**  
Before serving any letter, the server must verify that the requesting user is the owner of that letter. This check must happen server-side — client-side restrictions are trivially bypassed.


**2. Avoid exposing sequential integer IDs in URLs**  
Using predictable IDs makes enumeration trivial. Replace sequential integers with UUIDs or other non-guessable identifiers:

```
/letter/a3f9c2d1-4b5e-4f6a-9c8d-1e2f3a4b5c6d
```

Note: this is obscurity, not a fix on its own — authorisation checks are still required, but non-sequential IDs raise the bar significantly.

**3. Implement automated IDOR testing in your CI/CD pipeline**  
Tools like Burp Suite's Autorize extension or custom scripts can catch missing authorisation checks before they reach production.


