---
title: "Authentication"
layout: default
permalink: "/authentication"
nav_order: 6
---

# Authentication

Spacetime uses Private-Public Keypairs and [JSON Web Tokens (JWTs)](https://www.rfc-editor.org/rfc/rfc7519) to
authenticate with its services.

In order to authenticate, you'll need to generate a Private-Public keypair and send a self-signed x509 certificate containing the public key to Aalyria. You will then be provided with a `USER_ID` and a `KEY_ID` that can be used, together with your Private-Public Keypair, with [Spacetime SDK libraries](https://github.com/aalyria/api) or the [nbictl](https://github.com/aalyria/api/tree/main/tools/nbictl) command line utility.


## Creating Private-Public Keypairs

### Using `nbictl` 

```sh
$ nbictl generate-keys --org "my-organization.com"
private key is stored under: ~/.config/nbictl/keys/922e75e63659b29b76631275.key
certificate is stored under: ~/.config/nbictl/keys/922e75e63659b29b76631275.crt
```

### Using `Openssl`

```bash
# generate a private key of size 4096 and save it to priv_key.pem
openssl genrsa -out priv_key.pem 4096

# extract the public key and save it to an x509 certificate named
# pub_key.cer (with an expiration far into the future)
openssl req -new -x509 -key priv_key.pem -out pub_key.cer -days 36500
```

### Production Keypairs

In production, you'll want to create and store your keypair using a Trusted Platform Module (TPM)
to ensure that your private key cannot be exfiltrated. 
While the instructions for using TPM to create a use a keypair are platform-specific, you can use this document as a reference for how to manually create and sign the JWT tokens required to successfully authenticate with the NBI server.


## Connect to Spacetime APIs

### Using command line tool `nbictl` 

Once you have created private key and have shared the certificate with Aalyria, you can configure `nbictl` with the `set-config` command:

```sh
$ nbictl set-config --url "nbi.$INSTANCE_NAME.spacetime.aalyria.com:443"
$ nbictl set-config --key_id "$KEY_ID"
$ nbictl set-config --user_id "$USER_ID"
$ nbictl set-config --priv_key "$PRIVATE_KEY_FILE_NAME"
```

Once `nbictl` is configured, you can verify access to the Spacetime APIs by creating a request as follows:

```sh
$ nbictl list --type "NETWORK_NODE"
```

If successful, you should see a list of network nodes. The results returned will of course depend on the state of the Spacetime instance. It is normal for the results to be empty if no network nodes have yet been created in the instance.

### Programmatically with Java, Go or Python

Libraries and code samples are available for authenticating and interacting with Spacetime API in Java, Go and Python.

* **Java**:
  * [Auth Library](https://github.com/aalyria/api/tree/main/java/com/aalyria/spacetime/authentication)
  * [Code Samples](https://github.com/aalyria/api/tree/main/java/com/aalyria/spacetime/codesamples)
* **Go**
  * [Auth Library](https://github.com/aalyria/api/tree/main/auth)
  * [Open-source nbictl tool](https://github.com/aalyria/api/tree/main/tools/nbictl)
  * [CDPI Agent Reference Implementation](https://github.com/aalyria/api/tree/main/cdpi_agent)
* **Python**:
  * [Auth Library](https://github.com/aalyria/api/tree/main/py/authentication)
  * [Code Samples](https://github.com/aalyria/api/tree/main/py/authentication)

### Creating the gRPC request manually with `grpcurl`

To interact manually with the Spacetime API it's possible to use the [grpcurl](https://github.com/fullstorydev/grpcurl#installation) tool.

Example: list the methods available on the Spacetime NBI API.

```sh
grpcurl -H "Authorization: Bearer $SPACETIME_AUTH_JWT" \
        -H "Proxy-Authorization: Bearer $OIDC_TOKEN" \
        nbi.$INSTANCE_NAME.spacetime.aalyria.com:443 \
        list \
        aalyria.spacetime.api.nbi.v1alpha.NetOps
```

As described in the example above, to authenticate with Spacetime API it's necessary to include two *different* headers:

-  `Authorization`: A self-signed JWT Token called `SPACETIME_AUTH_JWT` 

-  `Proxy-Authorization`: A supplementary OIDC Token obtained after authenticating with Google Cloud using a self-signed JWT Token called `PROXY_AUTH_JWT`

#### Manually generate JWT Tokens

The requirements of each JWT are described in the table below.


| Field             | Spacetime JWT        | Proxy JWT                                                                 |
| ---------------   | ---------------------| ------------------------------------------------------------------------- |
| algorithm (`alg`) | `RS256`              | `RS256`                                                                   |
| issuer (`iss`)    | `$USER_ID`           | `$USER_ID`                                                                |
| subject (`sub`)   | `$USER_ID`           | `$USER_ID`                                                                |
| audience (`aud`)  | `$DOMAIN`            | `https://www.googleapis.com/oauth2/v4/token`                              |
| key ID (`kid`)    | `$KEY_ID`            | `$KEY_ID`                                                                 |
| lifetime (`exp`)  | Long-lived           | Short-lived (now + `1h`)                                                  |
| `target_audience` | `N/A`                | `60292403139-me68tjgajl5dcdbpnlm2ek830lvsnslq.apps.googleusercontent.com` |

A tool to generate self-signed JWT tokens is included in the Spacetime API public repository ([tools/generate_jwt](https://github.com/aalyria/api/tree/main/tools/generate_jwt)) and can be used as follow:

```sh
# Customer-specific details:
DOMAIN="${DOMAIN:?should be provided by your Aalyria contact}"
USER_ID="${USER_ID:?should be provided by your Aalyria contact}"
KEY_ID="${KEY_ID:?should be provided by your Aalyria contact}"
PRIV_KEY_FILE="/path/to/your/private/key/in/PKSC8/format.pem"

# Spacetime-specific details:
PROXY_AUDIENCE="https://www.googleapis.com/oauth2/v4/token"
PROXY_TARGET_AUDIENCE="60292403139-me68tjgajl5dcdbpnlm2ek830lvsnslq.apps.googleusercontent.com"

SPACETIME_AUTH_JWT=$(bazel run //tools/generate_jwt \
  --issuer "$USER_ID" --subject "$USER_ID" \
  --key-id "$KEY_ID" --private-key "$PRIV_KEY_FILE" \
  --audience "$DOMAIN")

PROXY_AUTH_JWT=$(bazel run //tools/generate_jwt \
  --issuer "$USER_ID" --subject "$USER_ID" \
  --key-id "$KEY_ID" --private-key "$PRIV_KEY_FILE"
  --audience "$PROXY_AUDIENCE" --target-audience "$PROXY_TARGET_AUDIENCE")
```

#### Manually obtain the OIDC Token

To obtain the OIDC Token you need to authenticate with Google Cloud API using the JWT Token `PROXY_AUTH_JWT` generated in previous step.

This can be done manually by using `curl` and `jq` commands as described in the example below.

```sh
curl -s \
      --data-urlencode "grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer" \
      --data-urlencode "assertion=${PROXY_AUTH_JWT}" \
      https://www.googleapis.com/oauth2/v4/token | jq -r '.id_token'
```
