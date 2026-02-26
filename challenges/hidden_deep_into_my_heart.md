# Hidden Deep Into My Heart

**Category:** Web Exploitation  
**Vulnerability:** Sensitive Information Disclosure via robots.txt (Credentials in Comments)  
**Difficulty:** Easy

---

## TL;DR

A web app exposes a hidden directory and a hardcoded password through its `robots.txt` file. The password works on the admin login page, granting direct access to the flag.

---

## Reconnaissance

### Web Application

The homepage has no visible functionality. Two parallel recon steps: source code review and directory fuzzing.

![IMG 1](/challenges/screenshots/hidden_deep_into_my_heart/image.png)


Source code review: nothing interesting.

### Directory Fuzzing

![IMG 2](/challenges/screenshots/hidden_deep_into_my_heart/image(1).png)

No significant findings from fuzzing directly on the root, except a robots.txt endpoint.

### robots.txt

![IMG 3](/challenges/screenshots/hidden_deep_into_my_heart/image(2).png)

Two findings:

1. A hidden directory: `/cupids_secret_vault`
2. A string that looks like a password: `cupid_arrow_2026!!!`

---

## Exploitation

### Step 1: Explore the Hidden Directory

Navigating to `/cupids_secret_vault` reveals a page. 
![IMG 4](/challenges/screenshots/hidden_deep_into_my_heart/image(3).png)


Fuzzing this directory:

![IMG 5](/challenges/screenshots/hidden_deep_into_my_heart/image(4).png)

Discovers: `/cupids_secret_vault/administrator`

### Step 2: Access the Login Page

`/cupids_secret_vault/administrator` presents a login form. Basic SQLi attempts fail.

![IMG 6](/challenges/screenshots/hidden_deep_into_my_heart/image(5).png)


### Step 3: Use the Leaked Credentials

The string found in `robots.txt` looks like a developer-left password. Trying:

- **Username:** `admin`
- **Password:** `cupid_arrow_2026!!!`

Login succeeds. Flag retrieved.

![IMG 7](/challenges/screenshots/hidden_deep_into_my_heart/image(6).png)

---

## Remediation

**1. Never put sensitive information in robots.txt**  
`robots.txt` is publicly accessible by design and indexed by search engines. It should only contain directives for web crawlers: never directory names you want to hide, and absolutely never credentials. Listing a directory in `robots.txt` to keep it hidden is security through obscurity and is actively counterproductive since it advertises the path to anyone who reads the file.

**2. Remove hardcoded credentials from source files and comments**  
Passwords, API keys, and secrets have no place in code, comments, config files, or any file that gets deployed to a web server. Use environment variables or a secrets manager instead.

**3. Enforce strong authentication on admin panels**  
Admin panels should require strong, unique passwords and ideally multi-factor authentication. A password accidentally left in a public file should not be sufficient to compromise the entire admin interface.

**4. Conduct regular secret scanning**  
Tools like `truffleHog`, `gitleaks`, or GitHub's built-in secret scanning can catch leaked credentials in codebases and config files before they reach production.

