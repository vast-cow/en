---
title: "Adding Authentication to an Existing Web App with Nginx + Authelia"
description: "Yes, given that condition, it's appropriate to completely rely on Nginx + an external authentication..."
pubDatetime: 2026-09-19T11:51:27.735Z
---

Yes, given that condition, it's appropriate to **completely rely on Nginx + an external authentication service for authentication, and have the app only receive the authenticated username in the header.**

## Recommended Configuration

The primary candidate is **Nginx + Authelia**.

```text
                   Internet
                      │
                 HTTPS :443
                      │
                ┌─────▼─────┐
                │   nginx   │
                │ TLS termination   │
                │ auth_request
                └─────┬─────┘
                      │
             Authentication │        Unauthenticated
        ┌─────────────┘           │
        ▼                         ▼
 ┌─────────────┐           ┌──────────────┐
 │  Authelia   │◀─────────▶│ auth.example │
 │ password/MFA│           │ Login screen │
 └─────────────┘           └──────────────┘
        │
   Authentication OK
        │
        ▼
 ┌───────────────────┐
 │ nginx             │
 │ Remote-User: foo  │
 │ is forcibly added     │
 └─────────┬─────────┘
           │ private network only
           ▼
 ┌───────────────────┐
 │ Existing App      │
 │ user = Remote-User│
 └───────────────────┘
```

Authelia officially supports Nginx's `auth_request`, and after successful authentication, it can return `Remote-User`, `Remote-Groups`, `Remote-Email`, etc. to Nginx. Nginx can receive the value with `auth_request_set` and pass it to the backend as a header. [Authelia][1]

### Why Authelia?

It aligns well with the current requirements.

| Requirement                      | Nginx + Authelia |
| -------------------------------- | ---------------- |
| Minimize changes to existing app | ◎                |
| Username + password              | ◎                |
| `Remote-User`                    | ◎ Native          |
| Continue using Nginx             | ◎                |
| Local user DB                    | ◎                |
| LDAP integration                | ◎                |
| Add MFA later                    | ◎                |
| Access control by group          | ◎                |
| Brute-force protection          | ◎                |
| Enable SSO                       | ◎                |
| Small-scale configuration       | ◎                |
| Expand to multiple apps          | ◎                |

Authelia's file backend uses Argon2id and provides temporary bans for users/IPs based on the number of failed authentication attempts. [Authelia][2]

---

# Changes on the App Side

In essence, the app needs to do this:

```text
Remote-User: alice
```

Treat this as:

```text
Currently logged-in user = alice
```

For example:

```pseudo
username = request.headers["Remote-User"]

if username is empty:
    return 401

user = find_or_create_user(username)
```

However, there is a **very important security boundary here.**

The app itself must unconditionally trust that:

> If `Remote-User` is present, the user is authenticated.

Therefore, **the app must not be accessible by bypassing Nginx.**

---

# Most Important Point: Do Not Expose the App Directly

For example, in Docker:

```yaml
app:
  expose:
    - "8080"
```

is okay, but:

```yaml
ports:
  - "8080:8080"
```

is not.

The configuration should only allow:

```text
Internet ──► nginx ──► app
                 │
                 └──► Authelia
```

The following is NOT allowed:

```text
Internet ──► nginx ──► app
   │
   └────────────────► app:8080   ← NG
```

If the latter is allowed, an attacker could bypass authentication by sending:

```http
GET /
Remote-User: admin
```

directly.

---

# Nginx Must Also Overwrite the Header

Do not forward the `Remote-User` sent from the browser as is.

The concept is:

```nginx
location / {
    auth_request /internal/authelia/authz;

    auth_request_set $authenticated_user $upstream_http_remote_user;

    proxy_set_header Remote-User       $authenticated_user;
    proxy_set_header X-Forwarded-User  $authenticated_user;

    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-Host  $host;

    proxy_pass http://app:8080;
}
```

The key is:

```nginx
proxy_set_header Remote-User $authenticated_user;
```

Even if the client sends:

```http
Remote-User: admin
```

Nginx will **replace it with the value obtained from the authentication service.**

Authelia's official Nginx configuration also uses:

```nginx
auth_request_set $user $upstream_http_remote_user;
proxy_set_header Remote-User $user;
```

[Authelia][1]

---

# Should I Use `Remote-User` or `X-Forwarded-User`?

In this case, **`Remote-User` is recommended**.

Authelia directly returns:

```text
Remote-User
Remote-Groups
Remote-Name
Remote-Email
```

[Authelia][1]

For example:

```http
Remote-User: tanaka
Remote-Email: tanaka@example.com
Remote-Groups: users,developers
```

The app generally only needs to look at:

```text
Remote-User
```

If you need to add `admin`, `editor`, or `viewer` in the future, you can use `Remote-Groups`.

---

# Authentication Flow

When a user accesses:

```text
https://app.example.com/foo
```

the following happens:

```text
1. Browser → nginx

2. nginx
   └─ auth_request → Authelia

3. Authelia
   ├─ session exists → OK
   └─ session does not exist → redirect to login

4. User
   username/password entry

5. Authelia
   └─ session cookie issued

6. Again, app.example.com/foo

7. nginx → Authelia
            ↓
          alice

8. nginx → App
   Remote-User: alice
```

Nginx's `auth_request` allows access if the authentication subrequest returns a 2xx status code and denies access if it returns 401/403. [Nginx][3]

---

# Authelia Side

If the number of users is small, you can store user information in a file.

The concept is:

```yaml
authentication_backend:
  file:
    path: /config/users.yml
    password:
      algorithm: argon2
      argon2:
        variant: argon2id
```

The current Authelia file backend recommends Argon2id as the default setting. [Authelia][2]

For example, register users:

```text
alice
bob
charlie
```

Instead of storing the passwords themselves, store:

```text
$argon2id$...
```

the hash.

---

# If Publishing to the Internet, Use `default deny`

It is a good idea for Authelia to have:

```yaml
access_control:
  default_policy: deny

  rules:
    - domain: app.example.com
      policy: one_factor
```

Authelia itself recommends `default_policy: deny`. [Authelia][4]

Therefore, even if the configuration is wrong, it will not be:

```text
No configuration → Public
```

but:

```text
No configuration → Denied
```

---

# You Can Start with Password-Only and Then Migrate to MFA

Initially, use:

```yaml
policy: one_factor
```

and only:

```text
username
password
```

Then, change to:

```yaml
policy: two_factor
```

to:

```text
password
+
TOTP / WebAuthn, etc.
```

Authelia has `one_factor` / `two_factor` as access control policies. [Authelia][4]

If you are placing the service on the public internet, it is worth considering configuring two-factor authentication (2FA) for administrators as well.

---

# Brute-Force Protection

This is a major advantage compared to simple Nginx Basic Auth.

Authelia has, for example:

```yaml
regulation:
  modes:
    - user
    - ip
  max_retries: 5
  find_time: 2m
  ban_time: 15m
```

as authentication attempt restrictions.

The current documentation also provides a mechanism to temporarily ban users/IPs based on the number of attempts to the username/password endpoint. [Authelia][5]

---

# Storage Configuration

## Small-Scale / Single Server

This is sufficient.

```text
nginx
Authelia
Existing App
SQLite
users.yml
```

Authelia can use a local SQLite storage. The official example has this configuration. [Authelia][6]

It becomes quite easy with Docker Compose.

```text
docker compose
├── nginx
├── authelia
└── app
```

*   volumes:

```text
authelia/config
authelia/db.sqlite3
```

---

## If HA is Needed

At that point, you can migrate to:

```text
        nginx
          │
   ┌──────┴──────┐
Authelia1    Authelia2
   │              │
   └──────┬───────┘
          │
     PostgreSQL
          +
        Redis
```

Authelia's session storage supports memory for single-instance and Redis / Redis Sentinel for HA, and Redis-based is recommended for HA purposes. [Authelia][7]

There is no need to do this from the beginning.

---

# TLS

This should be considered essential.

```text
https://app.example.com
https://auth.example.com
```

For example:

```text
Let's Encrypt
    ↓
nginx TLS termination
    ↓
internal docker network
```

Authelia itself requires HTTPS/WSS for both the authentication portal and forward authentication. [Authelia][8]

---

# Nginx Basic Auth as a Simpler Alternative

In fact, if you want to add almost nothing, you can also use:

```text
nginx
+
.htpasswd
```

```nginx
location / {
    auth_basic "Restricted";
    auth_basic_user_file /etc/nginx/users.htpasswd;

    proxy_set_header Remote-User $remote_user;

    proxy_pass http://app:8080;
}
```

Nginx's `$remote_user` contains the username from Basic Authentication. [Nginx][9]

Therefore, the extremely small configuration is also possible:

```text
Browser
  ↓
Basic Auth
  ↓
nginx
  ↓
Remote-User: alice
  ↓
App
```

### However

If it is for a general user service on the internet, I would treat this as a **temporary configuration**.

The reasons are:

*   Cannot create a login screen
*   Relies on the browser's Basic Auth UX
*   Explicit logout is difficult
*   No password change UI
*   Account management is `htpasswd`
*   No MFA
*   Weak lockout/authentication audit
*   Difficult for future SSO

If it is a tool used by only 3 administrators, it is rational, but if there are general users, Authelia is easier to use.

---

# Another Option: Authentik

Another viable option is:

```text
nginx + authentik
```

Authentik also has a **Forward Auth** for Nginx, so you don't need to proxy the existing app itself through authentik, and you can have:

```text
nginx → Authentik asks for authentication
nginx → existing app
```

For authenticated users:

```text
X-authentik-username
X-authentik-groups
X-authentik-email
X-authentik-name
X-authentik-uid
```

can be output. [authentik][11]

Furthermore, it is possible to convert to:

```text
Remote-User
```

with a custom header. [authentik][12]

However, for the current use case, Authentik is slightly more complex than Authelia.

---

# Summarizing the 3 Options

| Configuration                  | Ease of Use | Public-Facing | User Management | MFA | Suitable Cases     |
| ------------------------------ | ----------- | ------------- | --------------- | --- | ------------------ |
| Nginx Basic Auth               | ◎           | △             | △               | ×   | Few Users          |
| **Nginx + Authelia**           | ○           | ◎             | ○               | ◎   | **This Case**      |
| Nginx + Authentik              | △           | ◎             | ○               | ◎   | Full-fledged IAM/SSO |

Given the current requirements, **Nginx + Authelia + Remote-User** is the most balanced design.

---

# If You Were to Actually Set It Up, This Configuration

```text
                     Internet
                         │
                    TCP 80/443
                         │
                 ┌───────▼───────┐
                 │     nginx     │
                 │               │
                 │ TLS           │
                 │ auth_request  │
                 │ rate limit    │
                 │ access log    │
                 └───┬───────┬───┘
                     │       │
              /auth  │       │ /app
                     ▼       │
                ┌─────────┐  │
                │Authelia │  │
                └────┬────┘  │
                     │       │
                ┌────▼────┐  │
                │ users   │  │
                │ SQLite  │  │
                └─────────┘  │
                             │
            Remote-User: alice
                             │
                             ▼
                       ┌──────────┐
                       │   App    │
                       │ :8080    │
                       └──────────┘

nginx : public
Authelia : internal + only via nginx
App : internal only
SQLite/users.yml : filesystem only
```

## Things the App Side Should Implement at a Minimum

Authentication itself does not need to be implemented, but the following should be included:

1.  Deny access if `Remote-User` is missing
2.  Use `Remote-User` as the user ID
3.  Do not allow connections from outside Nginx
4.  Switch users based on `Remote-User` and **link all data access to that user**
5.  If username changes are allowed, consider designing with an immutable ID instead of the username.

In particular, **4 is important.** Nginx/Authelia only solves the **authentication** of "this person is alice."

```text
alice can only read alice's data
bob can only read bob's data
admin can use management functions
```

The app itself must guarantee this **authorization.**

---

### Conclusion

If the goal is a minimal configuration for publishing to the public, I recommend:

```text
Let's Encrypt
    ↓
nginx
    ↓ auth_request
Authelia
    ↓
Remote-User
    ↓
Existing App
```

**The changes on the app can be limited to "treating the trusted `Remote-User` as the current user," and password storage, login screens, sessions, MFA, and brute-force protection can be completely separated from the app.**

Next, it is possible to create a `compose.yml`, nginx.conf, Authelia configuration.yml, and users.yml that can be used to start `nginx + Authelia + existing app` with Docker Compose.
