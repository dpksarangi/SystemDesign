# OAuth 2.0

## 1. What OAuth Solves

OAuth 2.0 is primarily an **authorization/delegation framework**, not an authentication protocol.

It allows a client to obtain limited access to a resource.

Core roles:

```text
Resource Owner
      ↓
Authorization Server
      ↓
Access Token
      ↓
Resource Server
```

## 2. Authorization Code Flow

High level:

```text
User
 ↓
Client Application
 ↓
Authorization Server
 ↓
User authenticates/consents
 ↓
Authorization Code
 ↓
Client
 ↓
Token Endpoint
 ↓
Access Token + optionally Refresh Token
```

The access token is then sent to the resource server.

## 3. PKCE

PKCE protects the authorization code flow for public clients.

```text
code_verifier
      ↓
code_challenge
      ↓
Authorization Server
```

Later the client presents the `code_verifier` when exchanging the authorization code.

The attacker who steals only the authorization code cannot use it without the verifier.

## 4. Client Credentials Grant

For machine-to-machine communication:

```text
Service A
   │
   │ client_id + client_secret
   ▼
Authorization Server
   │
   ▼
Access Token
   │
   ▼
Service B
```

No human user is required.

## 5. Access Token

Represents authorization to access resources.

```http
Authorization: Bearer <access_token>
```

The token can be:

- JWT
- Opaque token

OAuth does not require JWT.

## 6. Refresh Token

A refresh token is used to obtain a new access token without repeating the full authorization flow.

```text
Refresh Token
      ↓
Authorization Server
      ↓
New Access Token
```

Typically:

```text
Access Token  → short-lived, used with APIs
Refresh Token → longer-lived, used with authorization server
```

## 7. Client Secret vs Refresh Token

```text
Client Secret
→ proves/helps authenticate the application

Refresh Token
→ represents an existing authorization that can be exchanged for new access tokens
```

They are both sensitive but have different semantics.

## 8. Scopes

Scopes limit what a token can do:

```text
orders:read
orders:write
profile:read
```

The resource server should enforce them.

## 9. OAuth vs API Key

| API Key | OAuth |
|---|---|
| Usually static | Token lifecycle |
| Simple | More protocol machinery |
| Client identity | Delegated authorization |
| Usually no consent flow | Can represent user consent |
| Rotation/revocation | Expiry/refresh/revocation |

## Interview Questions

- OAuth vs authentication?
- Authorization Code vs Client Credentials?
- Why PKCE?
- Access token vs refresh token?
- OAuth token vs JWT?
- Why should refresh tokens not be sent to APIs?
- What are scopes?
