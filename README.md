# CanvasPro API: Authentication and Go-Live Commands

This guide shows external clients how to authenticate against a CanvasPro server and trigger a scene or queue "go live" by name.

## Base URL

All endpoints below are reachable through the server's HTTPS reverse proxy on port `33333`, under the `/api` prefix:

```
https://<your-canvaspro-host>:33333/api
```

Replace `<your-canvaspro-host>` with the address of your CanvasPro instance. The server always presents a self-signed certificate, so your HTTP client will need to be configured to trust it (or skip verification).

## 1. Authenticate and get a token

CanvasPro uses OAuth2-style password authentication. Exchange a username/password for a JWT bearer token:

```bash
curl -X POST "https://<host>:33333/api/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=YOUR_USERNAME&password=YOUR_PASSWORD"
```

Response:

```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "bearer"
}
```

Send this token on every subsequent request as a header:

```
Authorization: Bearer <access_token>
```

### Token lifetime

Access tokens expire after 10 minutes. There are two ways to keep a client authenticated without asking for the username and password again each time. Pick whichever fits your integration.

**Option A: Roll the short-lived token**

Before the current token expires, exchange it for a new 10-minute token using the token itself, no credentials needed:

```bash
curl -X PUT "https://<host>:33333/api/update_token" \
  -H "Authorization: Bearer <access_token>"
```

Repeat this on an interval shorter than 10 minutes to stay authenticated indefinitely. This suits long-running processes that stay online and can refresh on a timer.

**Option B: Switch to a long-lived token**

Exchange the current token for one valid 24 hours:

```bash
curl -X GET "https://<host>:33333/api/generate_long_token" \
  -H "Authorization: Bearer <access_token>"
```

This returns a token valid for 24 hours, in the same `{"access_token", "token_type"}` shape. Before it expires, log in again with `/token` using the username and password to get a fresh token. This suits clients that want to touch the token endpoint as little as possible.

## 2. Send a "go live" command by name

Once authenticated, trigger any scene or queue live using its display name (`Lower Third 1` below is just an example, use the name of your own scene or queue):

```bash
curl -X GET "https://<host>:33333/api/dynamic_engine/go_live?name=Lower%20Third%201" \
  -H "Authorization: Bearer <access_token>"
```

- Matching is case-insensitive and ignores leading/trailing whitespace.
- If multiple items share the same name, all of them are triggered live.
- If nothing matches, the server responds `404 Not Found`.

## 3. Example client (JavaScript)

```js
const HOST = "https://<your-canvaspro-host>:33333";

async function login(username, password) {
  const body = new URLSearchParams({ username, password });
  const res = await fetch(`${HOST}/api/token`, {
    method: "POST",
    headers: { "Content-Type": "application/x-www-form-urlencoded" },
    body,
  });
  if (!res.ok) throw new Error(`Login failed: ${res.status}`);
  const { access_token } = await res.json();
  return access_token;
}

async function goLiveByName(token, name) {
  const res = await fetch(
    `${HOST}/api/dynamic_engine/go_live?name=${encodeURIComponent(name)}`,
    { headers: { Authorization: `Bearer ${token}` } }
  );
  if (!res.ok) throw new Error(`go_live failed: ${res.status}`);
}

const token = await login("YOUR_USERNAME", "YOUR_PASSWORD");
await goLiveByName(token, "Lower Third 1");
```

## Error responses

| Status | Meaning |
|---|---|
| 401 Unauthorized | Missing/invalid/expired token, or bad username/password on `/token` |
| 400 Bad Request | Missing or conflicting query parameters (e.g. neither `did` nor `name` provided) |
| 404 Not Found | No scene or queue matches the given name |

## License

This documentation is licensed under [CC BY 4.0](LICENSE). CanvasPro itself is closed source, proprietary commercial software; this license applies only to the contents of this repository (the API documentation), not to the CanvasPro product or its source code.
