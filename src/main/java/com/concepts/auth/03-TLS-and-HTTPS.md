# TLS & HTTPS

## 1. What TLS Provides

TLS provides:

```text
Confidentiality
Integrity
Server authentication
```

HTTPS = HTTP over TLS.

## 2. High-Level Handshake

```text
Client                          Server
  │                               │
  │ ClientHello                   │
  │──────────────────────────────>│
  │                               │
  │       Certificate             │
  │<──────────────────────────────│
  │                               │
  │ Key exchange                  │
  │──────────────────────────────>│
  │                               │
  │   Session keys established    │
  │<─────────────────────────────>│
  │                               │
  │ Encrypted HTTP                │
  │<─────────────────────────────>│
```

Modern TLS commonly uses asymmetric cryptography for authentication/key establishment and symmetric AEAD encryption for application data.

## 3. Certificate

The certificate helps the client verify:

```text
"I am really talking to api.example.com."
```

The CA chain establishes trust.

## 4. Symmetric vs Asymmetric

Asymmetric:

```text
Private key → sign
Public key  → verify
```

Symmetric:

```text
Shared session key → encrypt/decrypt
```

Symmetric encryption is used for bulk traffic because it is efficient.

## 5. Forward Secrecy

Modern ephemeral key exchange can provide forward secrecy.

If a server's long-term private key is compromised later, previously recorded TLS traffic should not automatically become decryptable.

## 6. TLS Connection Reuse

TLS is normally established for a connection, not independently for every HTTP request.

HTTP/2 can multiplex many requests over one TLS connection.

TLS session resumption makes reconnects cheaper.

## 7. TLS vs Authentication

TLS protects the communication channel.

Application authentication answers:

```text
Who is the caller?
```

Examples:

- Password
- Session cookie
- API key
- OAuth token
- mTLS certificate

You commonly need both:

```text
HTTPS
  +
Application authentication
```

## 8. TLS Interception / JMeter

A test proxy can act as a trusted TLS interception point:

```text
Browser
   ↓ TLS
JMeter
   ↓ TLS
Real Server
```

The browser trusts a JMeter-generated certificate because the JMeter CA certificate was installed as trusted.

This is not breaking TLS cryptography; it changes the trust path.

## Interview Questions

- Why does HTTPS need certificates?
- Why not encrypt all HTTP data using RSA?
- Why use symmetric encryption after the handshake?
- What is forward secrecy?
- Does TLS happen for every HTTP request?
- What does a CA actually prove?
- How does an HTTPS proxy inspect traffic?
