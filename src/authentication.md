---
title: "Authentication"
layout: default
permalink: "/authentication"
nav_order: 5
---

# Authentication

Spacetime uses signed [JSON Web Tokens (JWTs)](https://www.rfc-editor.org/rfc/rfc7519) to
authenticate with the CDPI service. The JWT needs to be signed using an RSA
private key with a corresponding public key that's been shared - inside of a
self-signed x509 certificate - with the Aalyria team.

## Non-Production Authorization

Follow these steps for creating JWTs to be used for interacting with Spacetime's 
APIs in demo (non-production) environments.

### Creating a Test Keypair

For non-production purposes, you can generate a valid key using the `openssl` tool:

```bash
# generate a private key of size 4096 and save it to agent_priv_key.pem
openssl genrsa -out agent_priv_key.pem 4096
# extract the public key and save it to an x509 certificate named
# agent_pub_key.cer (with an expiration far into the future)
openssl req -new -x509 -key agent_priv_key.pem -out agent_pub_key.cer -days 36500
```

### Creating Test JWTs

Once you have your private key and you've shared the public key with Aalyria,
you can create a signed JWT to authorize the agent. To facilitate testing agent
authorization, we have a small [Go program](https://github.com/aalyria/api/tree/main/tools/generate_jwt) 
that can be used to generate signed JWTs.

Authorization requires two *different* JWTs:

- A Spacetime JWT, passed in the `Authorization` header / `--authorization-jwt`
  flag and referenced below as `SPACETIME_AUTH_JWT`

- A supplementary JWT consumed by the CDPI service's secure proxy, passed in
  the `Proxy-Authorization` header / `--proxy-authorization-jwt` flag and
  referenced below as `PROXY_AUTH_JWT`

NOTE: Your contact on the Aalyria team will be able to provide the full values
for the below `$AGENT_EMAIL`, `$CDPI_DOMAIN`, and `$AGENT_PRIV_KEY_ID`
variables.

Using the included program to create these tokens is simple.

```bash
# Customer-specific details:
CDPI_DOMAIN="${CDPI_DOMAIN:?should be provided by your Aalyria contact}"
AGENT_EMAIL="${AGENT_EMAIL:?should be provided by your Aalyria contact}"
AGENT_PRIV_KEY_ID="${AGENT_PRIV_KEY_ID:?should be provided by your Aalyria contact}"
# This is the "agent_priv_key.pem" file from above:
AGENT_PRIV_KEY_FILE="/path/to/your/private/key/in/PKSC8/format.pem"

# Spacetime-specific details:
PROXY_AUDIENCE="https://www.googleapis.com/oauth2/v4/token"
PROXY_TARGET_AUDIENCE="60292403139-me68tjgajl5dcdbpnlm2ek830lvsnslq.apps.googleusercontent.com"

SPACETIME_AUTH_JWT=$(bazel run //cdpi_agent/cmd/generate_jwt \
  -- \
  --issuer "$AGENT_EMAIL" \
  --subject "$AGENT_EMAIL" \
  --audience "$CDPI_DOMAIN" \
  --key-id "$AGENT_PRIV_KEY_ID" \
  --private-key "$AGENT_PRIV_KEY_FILE")

PROXY_AUTH_JWT=$(bazel run //cdpi_agent/cmd/generate_jwt \
  -- \
  --issuer "$AGENT_EMAIL" \
  --subject "$AGENT_EMAIL" \
  --audience "$PROXY_AUDIENCE" \
  --target-audience "$PROXY_TARGET_AUDIENCE" \
  --key-id "$AGENT_PRIV_KEY_ID" \
  --private-key "$AGENT_PRIV_KEY_FILE")
```

Use these 2 JWTs when invoking the methods in the NBI or CDPI.

## Production Authorization

In production, you'll likely want to use a Trusted Platform Module (TPM) to
keep your private key secure when signing the authorization JWTs. While the
platform-specific details of generating signed JWTs using a TPM are outside the
scope of this project, the fields for both JWTs are listed below:

| Field             | Spacetime JWT                   | Proxy JWT                                                                 |
| ---------------   | ------------------------------- | -----------------------------------------------------------------------   |
| algorithm (`alg`) | `RS256`                         | `RS256`                                                                   |
| issuer (`iss`)    | `$AGENT_EMAIL`                  | `$AGENT_EMAIL`                                                            |
| subject (`sub`)   | `$AGENT_EMAIL`                  | `$AGENT_EMAIL`                                                            |
| audience (`aud`)  | `$CDPI_DOMAIN`                  | `https://www.googleapis.com/oauth2/v4/token`                              |
| key ID (`kid`)    | `$AGENT_PRIV_KEY_ID`            | `$AGENT_PRIV_KEY_ID`                                                      |
| lifetime (`exp`)  | Long-lived                      | Short-lived (now + `1h`)                                                  |
| `target_audience` | `N/A`                           | `60292403139-me68tjgajl5dcdbpnlm2ek830lvsnslq.apps.googleusercontent.com` |