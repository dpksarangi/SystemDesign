# Username, Password & Sessions

## 1. Password Storage

Passwords should NOT be stored encrypted for normal login verification.

Store a salted password hash:

```text
password
   ↓
random salt
   ↓
Argon2id / bcrypt / scrypt / PBKDF2
   ↓
password hash
```

At login:

```text
entered password + stored salt
            ↓
        password hash
            ↓
       compare hashes
```

The server does not decrypt the password.

## 2. Why Salt?

Without salt:

```text
password123 → same hash everywhere
```

With random salt:

```text
password123 + saltA → hashA
password123 + saltB → hashB
```

Salt defeats precomputed/rainbow-table attacks and prevents identical passwords from producing identical hashes.

## 3. Online vs Offline Brute Force

Online:

```text
Attacker → Login API → password attempt
```

Controls:

- Rate limiting
- Account lockout
- CAPTCHA/risk controls
- MFA

Offline:

```text
Database breach
     ↓
Attacker gets password hashes
     ↓
Guess passwords locally
```

Login attempt limits do not protect against offline cracking. Slow password hashing is important.

## 4. Session-Based Authentication

```text
POST /login
username + password
       ↓
password verification
       ↓
create session
       ↓
Set-Cookie: session_id=abc123
```

Later:

```http
GET /profile
Cookie: session_id=abc123
```

Server:

```text
session_id
    ↓
session store
    ↓
userId + session metadata
```

## 5. Distributed Sessions

With multiple application servers:

```text
             Load Balancer
              /                       ↓            ↓
         Server A      Server B
             \            /
              \          /
                Redis
```

A shared session store avoids requiring the user to return to the same server.

Sticky sessions are another option, but introduce coupling to a particular server.

## 6. Cookie Security

Important flags:

- `HttpOnly` — JavaScript cannot directly read the cookie.
- `Secure` — send only over HTTPS.
- `SameSite` — controls cross-site cookie sending.

## 7. Session Threats

- Session theft
- Session fixation
- CSRF
- Excessive session lifetime
- Missing logout/revocation
- Insecure cookies

## Session vs JWT

| Session | JWT |
|---|---|
| Server-side state | Usually self-contained |
| Easy revocation | Revocation is harder |
| Small cookie/session ID | Token carries claims |
| Shared store often needed | Local validation possible |
| Good for browser sessions | Common for APIs |

## Interview Questions

- Why use Redis for sessions?
- Sticky sessions vs shared session store?
- What is session fixation?
- HttpOnly vs Secure?
- SameSite and CSRF?
- Session vs JWT?
