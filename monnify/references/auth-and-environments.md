# Authentication and environments

## Credentials

Get them from the Monnify dashboard under **Developer → API Keys & Contracts** (some dashboard versions call it Settings → API Keys & Webhooks).

| Credential | Looks like | Where it is used |
|---|---|---|
| API Key | `MK_TEST_XXXXXXXXXX` (sandbox) / `MK_PROD_...` (live) | Login (server), Checkout SDK (frontend) |
| Secret Key | 32-character string | Login (server only), webhook signature verification |
| Contract Code | 10-digit number | Initialize transaction, reserved accounts, invoices, SDK |
| Wallet account number | 10 digits | Source account for transfers, wallet balance |

The dashboard toggle (top right) switches between Test Mode and Live Mode; each mode has its own keys.

## Base URLs

| Environment | Base URL |
|---|---|
| Sandbox | `https://sandbox.monnify.com` |
| Live | `https://api.monnify.com` |

Going live means swapping **all** of: base URL, API Key, Secret Key, Contract Code, wallet account number and webhook URLs.

## Getting an access token

```
POST /api/v1/auth/login
Authorization: Basic <base64(API_KEY + ":" + SECRET_KEY)>
(no body)
```

Response:

```json
{
  "requestSuccessful": true,
  "responseMessage": "success",
  "responseCode": "0",
  "responseBody": { "accessToken": "eyJhbGciOi...", "expiresIn": 3567 }
}
```

`expiresIn` is in seconds (about an hour). Cache the token and refresh shortly before it expires, or on the first 401.

### Node.js (fetch, no dependencies)

```js
const BASE = process.env.MONNIFY_BASE_URL ?? "https://sandbox.monnify.com";
let cached = { token: null, expiresAt: 0 };

export async function monnifyToken() {
  if (cached.token && Date.now() < cached.expiresAt - 60_000) return cached.token;
  const basic = Buffer.from(
    `${process.env.MONNIFY_API_KEY}:${process.env.MONNIFY_SECRET_KEY}`
  ).toString("base64");
  const res = await fetch(`${BASE}/api/v1/auth/login`, {
    method: "POST",
    headers: { Authorization: `Basic ${basic}` },
  });
  const json = await res.json();
  if (!json.requestSuccessful) throw new Error(`Monnify auth failed: ${json.responseMessage}`);
  cached = {
    token: json.responseBody.accessToken,
    expiresAt: Date.now() + json.responseBody.expiresIn * 1000,
  };
  return cached.token;
}

export async function monnify(path, { method = "GET", body } = {}) {
  const res = await fetch(`${BASE}${path}`, {
    method,
    headers: {
      Authorization: `Bearer ${await monnifyToken()}`,
      "Content-Type": "application/json",
    },
    body: body ? JSON.stringify(body) : undefined,
  });
  const json = await res.json();
  if (!json.requestSuccessful) {
    const err = new Error(json.responseMessage);
    err.code = json.responseCode;
    err.status = res.status;
    throw err;
  }
  return json.responseBody;
}
```

### Python (requests)

```python
import base64, os, time, requests

BASE = os.getenv("MONNIFY_BASE_URL", "https://sandbox.monnify.com")
_cache = {"token": None, "exp": 0}

def token():
    if _cache["token"] and time.time() < _cache["exp"] - 60:
        return _cache["token"]
    basic = base64.b64encode(
        f'{os.environ["MONNIFY_API_KEY"]}:{os.environ["MONNIFY_SECRET_KEY"]}'.encode()
    ).decode()
    r = requests.post(f"{BASE}/api/v1/auth/login", headers={"Authorization": f"Basic {basic}"}, timeout=30)
    data = r.json()
    if not data.get("requestSuccessful"):
        raise RuntimeError(data.get("responseMessage"))
    _cache.update(token=data["responseBody"]["accessToken"],
                  exp=time.time() + data["responseBody"]["expiresIn"])
    return _cache["token"]

def monnify(method, path, **kw):
    r = requests.request(method, f"{BASE}{path}",
                         headers={"Authorization": f"Bearer {token()}"}, timeout=30, **kw)
    data = r.json()
    if not data.get("requestSuccessful"):
        raise RuntimeError(f'{data.get("responseCode")}: {data.get("responseMessage")}')
    return data["responseBody"]
```

The official Python library is `pip install monnify-python` (`from monnify import Monnify; Monnify(API_KEY=..., SECRET_KEY=..., ENV="SANDBOX")`).

## Common auth failures

- **401 with sandbox keys** → the code is pointing at `api.monnify.com`. Check `MONNIFY_BASE_URL`.
- **401 on a business endpoint** → token expired or the header says `Basic` instead of `Bearer`.
- **"Invalid contract code"** → the contract code belongs to the other mode or another merchant.
- Trailing whitespace or a newline in the env var breaks the Base64 string; trim it.
