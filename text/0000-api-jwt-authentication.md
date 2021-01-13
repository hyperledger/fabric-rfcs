---
layout: default
title: JWT Authentication for Fabric APIs
nav_order: 3
---

- Feature Name: api_jwt_authentication
- Start Date: 2021-01-11
- RFC PR:
- Fabric Component: orderer, peer
- Fabric Issue:

# Summary

This document proposes an OAuth2 compatible authentication mechanism for MSP
identities that produces authorization tokens suitable for use with APIs
exposed by Fabric.

# Motivation

With the introduction of the channel participation API, the Fabric orderer
exposes administrative operations that are authorized based on mutual TLS
authentication. This approach was chosen as it protects the operations while
maintaining conventional REST interaction patterns without explicit message
hashing and signing.

There are two minor issues with this approach:

1. The client CA certificate used to control access to the channel
   participation API is disconnected from the orderer's local MSP.
2. It's often difficult for web applications to connect to servers with TLS
   mutual authentication.

This goal of this proposal is to describe an authentication flow based on
[RFC7523][RFC7523] that can be used to obtain an access token for the
channel participation API from an MSP identity.

# Guide-level Explanation

Fabric supports an authentication mechanism based on OAuth 2.0 and OpenID
connect that returns an access token representing an MSP identity. This access
token can then be used to interact with the channel participation API using
conventional `Authorization` headers instead of TLS mutual authentication.

## OpenID Connect Authentication

OpenID Connect is an identity layer that sits on top of the OAuth 2.0
protocol. Part of the specification describes how to verify the identify of an
end-user by authenticating to an authorization server.

The [`private_key_jwt`][private_key_jwt] client authentication method enables
a client to use a private key associated with a registered public key to sign
an authentication token and assert its own identity. This `client_assertion`
can then be presented to a resource server's `/token` endpoint to acquire an
access token. The token lifetime is returned along with the access token in
the response.

## Access Token

An access token is a credential that authorizes an application to access
protected resources. The access tokens issued by Fabric are not refreshable
and expire after a short period of time. The token itself is a random string
that references the serialized MSP identity used to obtain the token.

It's important to note that access tokens are _bearer_ tokens. These tokens
must be kept confidential and only used over secure transports.

## Using Access Tokens

Once an access token has been obtained, clients may present it on requests to
the channel participation API in the `Authorization` header. The server will
validate this token and make access control decisions based on the associated
identity.

# Reference-level Explanation

For the proposed authentication flow, a `/token` endpoint will be introduced
to the orderer HTTP server. The endpoint will require the use of TLS to help
ensure confidentiality.

## Authentication

The client authentication mechanism is based on the OAuth 2.0 specifications.
Section [4.2][RFC7521-4.2] of [RFC7521][RFC7521] defines how to present
`client_assertion` tokens to OAuth token endpoints and section
[2.2][RFC7523-2.2] of [RFC7523][RFC7523] defines the required parameters.

At a high level, authentication involves a `POST` request to the token
endpoint with a number of `application/x-www-form-urlencoded` parameters. The
parameters are:

- `grant_type` - the value `client_credentials`
- `client_assertion_type` - the value `urn:ietf:params:oauth:client-assertion-type:jwt-bearer`
- `client_assertion` - the value of the authentication token

When requests are received, the endpoint handler will examine the form
parameters and reject any requests that arrive with a grant type other than
`client_credentials` or an assertion type other than the JWT bearer URN with
`400 Bad Request`.

### Building the Authentication Token

The token used for authentication is a `client_assertion` JWT and consists of
three sections: the Javascript Object Signing and Encryption (JOSE) Header,
Claims, and the Javascript Web Signature (JWS) signature.

#### JOSE Header

The header in the `client_assertion` token indicates the type of the token,
the algorithm used to sign the token, and a reference to the public key that
can be used to verify the token signature. In Fabric, this means that `typ`
must be set to `JWT`, `alg` must be `ES256` for ECDSA-P256 certificates (the
only type we'll support initially), and `x5c` must be set to the certificate
or certificate chain of the client in the form described by
[RFC7517][RFC7517].

The following is an example of a valid header that includes an ECDSA-P256
certificate:

```json
{
  "alg": "ES256",
  "typ": "JWT",
  "x5c": [ "MIICPTCCAeSgAwIBAgICBnowCgYIKoZIzj0EAwIwdTELMAkGA1UEBhMCVVMxCTAHBgNVBAgTADEWMBQGA1UEBxMNU2FuIEZyYW5jaXNjbzEbMBkGA1UECRMSR29sZGVuIEdhdGUgQnJpZGdlMQ4wDAYDVQQREwU5NDAxNjEWMBQGA1UEChMNQ29tcGFueSwgSU5DLjAeFw0yMTAxMTIyMDUyMTNaFw0zMTAxMTIyMDUyMTNaMHUxCzAJBgNVBAYTAlVTMQkwBwYDVQQIEwAxFjAUBgNVBAcTDVNhbiBGcmFuY2lzY28xGzAZBgNVBAkTEkdvbGRlbiBHYXRlIEJyaWRnZTEOMAwGA1UEERMFOTQwMTYxFjAUBgNVBAoTDUNvbXBhbnksIElOQy4wWTATBgcqhkjOPQIBBggqhkjOPQMBBwNCAASEC/cWPHMJ+EslGw8dzcNINcJWE2H5eMUud1Yv+I+FmbngLabxCoHoEhnoASvyR4LseRfUrCcK7158nF74hoTho2QwYjAOBgNVHQ8BAf8EBAMCB4AwHQYDVR0lBBYwFAYIKwYBBQUHAwIGCCsGAQUFBwMBMA4GA1UdDgQHBAUBAgMEBjAhBgNVHREEGjAYhwR/AAABhxAAAAAAAAAAAAAAAAAAAAABMAoGCCqGSM49BAMCA0cAMEQCIAnkSckDXh/fDEg+qzKavblyc+wW8p9rGRvgac+i7KuZAiBMJxvuT1EOGNIL2ELFzK7iiP8+GKTPxpFqKHzp8gu2zA==" ]
}
```

#### Claims

The claims in the `client_assertion` token must include the following fields:

- `aud` - the intended "audience" for the claim and should be the URL of the of the token endpoint
- `iss` - the `client_id` associated with the certificate
- `sub` - the `client_id` associated with the certificate
- `exp` - the token expiration time expressed as the number of seconds since the Unix Epoch
- `iat` - the time the assertion was created expressed as the number of seconds since the Unix Epoch
- `jti` - a unique token identifier that helps prevent replay attacks

The `aud` value provided by the client must match the URL of the token
endpoint. Because of proxying, the expected value(s) may need to be configured
on the server.

The `iss` and `sub` fields declare the `client_id` that is being asserted.
Both fields must contain the same value and should be the _MSPID_ that issued
the certificate. This information allows us to validate the certificate
against the correct MSP.

The `exp` time in the token should be checked against the local clock. If the
expiry is in the past, the assertion must not be accepted. If the expiry time
is more than some threshold in the future, the assertion must not be accepted.

The `iat` time in the token should be checked against the local clock. If the
token was issued more than some threshold in the past, the assertion must not
be accepted. The window between `iat` and `exp` should represent a reasonable
window of time for authentication - typically less than a minute or two.

The `jti` is generated by the client and is intended to act as a nonce to
prevent replays. Prior to returning an `access_token` to the client, the
server will save a reference to the `jti` value in memory until the token
expires at `exp`. If a `client_assertion` is received with a previously
observed `jti`, the assertion token must not be accepted.


The following is an example of valid claims for an `mspid` identity
assertion to the `/token` endpoint at `orderer.example.com`.

```json
{
  "aud": [
    "https://orderer.example.com/token"
  ],
  "exp": 1610484793,
  "iat": 1610484733,
  "iss": "mspid",
  "jti": "de3e8a0a-c137-4de0-a16d-465fe01a1bde",
  "sub": "mspid"
}
```

#### JWS Signature

After creating the header and claims objects, a signature needs to be
generated over the two objects. This is done by base64 URL encoding the UTF-8
representation of the header and claims, joining them by a period (`.`),
generating a signature using the algorithms defined in [RFC7518][RFC7518]. The
resulting signature is then base64 URL encoded. The complete token is formed
by joining the encoded header, claims, and signature with periods (`.`).

The algorithm claimed in the header must be appropriate for the certificate
included in the header and must be the one used to generate the signature. The
only algorithm we will support initially will be `ES256` using for ECDSA-P256
private keys.

#### Example `client_assertion` JWT

```
eyJhbGciOiJFUzI1NiIsInR5cCI6IkpXVCIsIng1YyI6WyJNSUlDUFRDQ0FlU2dBd0lCQWdJQ0Jub3dDZ1lJS29aSXpqMEVBd0l3ZFRFTE1Ba0dBMVVFQmhNQ1ZWTXhDVEFIQmdOVkJBZ1RBREVXTUJRR0ExVUVCeE1OVTJGdUlFWnlZVzVqYVhOamJ6RWJNQmtHQTFVRUNSTVNSMjlzWkdWdUlFZGhkR1VnUW5KcFpHZGxNUTR3REFZRFZRUVJFd1U1TkRBeE5qRVdNQlFHQTFVRUNoTU5RMjl0Y0dGdWVTd2dTVTVETGpBZUZ3MHlNVEF4TVRJeU1EVXlNVE5hRncwek1UQXhNVEl5TURVeU1UTmFNSFV4Q3pBSkJnTlZCQVlUQWxWVE1Ra3dCd1lEVlFRSUV3QXhGakFVQmdOVkJBY1REVk5oYmlCR2NtRnVZMmx6WTI4eEd6QVpCZ05WQkFrVEVrZHZiR1JsYmlCSFlYUmxJRUp5YVdSblpURU9NQXdHQTFVRUVSTUZPVFF3TVRZeEZqQVVCZ05WQkFvVERVTnZiWEJoYm5rc0lFbE9ReTR3V1RBVEJnY3Foa2pPUFFJQkJnZ3Foa2pPUFFNQkJ3TkNBQVNFQy9jV1BITUorRXNsR3c4ZHpjTklOY0pXRTJINWVNVXVkMVl2K0krRm1ibmdMYWJ4Q29Ib0Vobm9BU3Z5UjRMc2VSZlVyQ2NLNzE1OG5GNzRob1RobzJRd1lqQU9CZ05WSFE4QkFmOEVCQU1DQjRBd0hRWURWUjBsQkJZd0ZBWUlLd1lCQlFVSEF3SUdDQ3NHQVFVRkJ3TUJNQTRHQTFVZERnUUhCQVVCQWdNRUJqQWhCZ05WSFJFRUdqQVlod1IvQUFBQmh4QUFBQUFBQUFBQUFBQUFBQUFBQUFBQk1Bb0dDQ3FHU000OUJBTUNBMGNBTUVRQ0lBbmtTY2tEWGgvZkRFZytxekthdmJseWMrd1c4cDlyR1J2Z2FjK2k3S3VaQWlCTUp4dnVUMUVPR05JTDJFTEZ6SzdpaVA4K0dLVFB4cEZxS0h6cDhndTJ6QT09Il19.eyJhdWQiOlsiYXVkdXJsIl0sImV4cCI6MTYxMDQ4NDc5MywiaWF0IjoxNjEwNDg0NzMzLCJpc3MiOiJjbGllbnRpZCIsImp0aSI6ImRlM2U4YTBhLWMxMzctNGRlMC1hMTZkLTQ2NWZlMDFhMWJkZSIsInN1YiI6ImNsaWVudGlkIn0.QXuDTU0GdNeG_ejB7wAkhLKvQKX27qE8_hayqmGnddzqCZ400cX7fV_tBd7-2LtgXbOatK5ifxMo7c81ItgQlQ
```

### Validating the `client_assertion` JWT

A number of libraries exist in `go` to process JWT tokens so there's little
reason to roll our own. For example, the [`go-jose.v2`][go-jose] package
implemented by square looks to be of high quality, supports `x5c` certificate
chains, and has an active community.

After using a library to verify that a token is well formed, not expired, and
valid, the claims can be evaluated. This is where we check that the `aud`
claim matches our token endpoint, that `iss` and `sub` are equal and contain a
known MSP identifier, and that the MSP considers the certificate (or chain)
valid.  Finally, we need to ensure that the `jti` has not already been
observed by this server.

Once these checks have been performed, we should be convinced that the
authentication token is valid, the certificate is valid, and that this
assertion will only be honored once in the window between `iat` and `exp`.

### Issuing the `access_token`

In response to a successful `client_assertion`, the token endpoint generates
an OAuth2 compatible JSON response that contains an `access_token`,
`token_type`, and `expires_in` properties. The response should not include a
`refresh_token` as the client can self-refresh by re-asserting its identity.

Since we only require our tokens to be valid for the current server, we don't
need to use a JWT for the `access_token`. For the sake of simplicity, we will
use a base64 encoded token that consists 32 bytes of random data; this is
equivalent to a session ID.

The `token_type` should be set to `bearer` and the `expires_in` value should
be some small, configurable duration that defaults to 300 seconds or less.

This generated token needs to be stored in memory and associated with the
certificate extracted from the `client_assertion`, the MSP ID from the
`client_id`, and the expiration time of the `access_token`. An asynchronous
task will be scheduled to reap expired tokens.

## Authorization

When a protected endpoint is accessed by a client, `http.Handler` middleware
will intercept the request to perform authorization. First, the middleware
will check for a bearer token in the `Authorization` header. If the header
does not contain a token, the request will be rejected with a `401
Unauthorized`.

Once we have the token, the we will look in the `access_token` table for the
certificate and expiry information associated with the token. If the entry is
not found, the request will be rejected with a `403 Forbidden`. If the
`access_token` is found but has expired, the request will be rejected with a
`403 Forbidden`.

At this point we should have the certificate associated with the client and
the MSP responsible for validating it. With these two bits of information, we
can check if the identity represented by the `access_token` is authorized to
perform the requested operation.

# Drawbacks

The major drawback to this proposal is that it introduces yet another
authentication mechanism to Fabric. That said, this mechanism is based on well
known and widely adopted standards with first-class support in many existing
libraries.

# Rationale and Alternatives

> Why is this design the best in the space of possible designs?

It may not be the best, but it's likely better than a one-off solution. This
design builds on existing standards and seems flexible enough to support our
current and future authentication and authorization requirements. The
authentication mechanism described here can also be applied to gRPC services
to support activities like chaincode install and query operations.

> What other designs have been considered and what is the rationale for not
> choosing them?

We considered using various forms of basic authentication that relied on
server side configuration and rejected these as they were too static and still
did not make use of the local MSP.

We also looked at the token implementation used by the Fabric-CA and rejected
that due to its complexity and inflexibility.

> What is the impact of not doing this?

If we decide not to do this, we will continue to rely on mutual TLS for
authentication using TLS certificates instead of MSP identities. This
complicates client development and require us to maintain separate endpoints
for channel participation and operations.

# Prior art

See [Rationale and Alternatives][#rationale-and-alternatives].

# Testing

Explicit integration tests that exercise JWT based authentication and access
token authorization will be required. These tests will target the HTTP server
that hosts the channel participation API.

# Dependencies

There are no known dependencies for this work.

# Unresolved questions

- What parts of the design do you expect to resolve through the RFC process
  before this gets merged?
- What parts of the design do you expect to resolve through the implementation
  of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be
  addressed in the future independently of the solution that comes out of this
  RFC?

[RFC7517]: https://tools.ietf.org/html/rfc7517
[RFC7518]: https://tools.ietf.org/html/rfc7518
[RFC7521-4.2]: https://tools.ietf.org/html/rfc7521#section-4.2
[RFC7521]: https://tools.ietf.org/html/rfc7521
[RFC7523-2.2]: https://tools.ietf.org/html/rfc7523#section-2.2
[RFC7523]: https://tools.ietf.org/html/rfc7523
[go-jose]: https://gopkg.in/square/go-jose.v2/jwt
[private_key_jwt]: https://openid.net/specs/openid-connect-core-1_0.html#ClientAuthentication
