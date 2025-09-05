---
title: "Defense Against Stolen OAuth 2.0 Tokens"
description: >-
  An examination of recent high-profile token compromise attacks against
  OAuth 2.0 and the unique challenges of defending against them.
tags:
  - security
---

## In the News

HTTP authentication tokens are an attractive target for hackers. Stealing an access token allows an
adversary to compromise a user account without having to break passwords or two-factor authentication.
In a [recent attack] on content creator Linus Media Group, an employee was tricked into executing a
malicious email attachment that copied browsing data off their computer. Using the access tokens,
the attacker was able to create YouTube livestreams, delete videos, and edit profile information.

It is difficult to detect and stop token-based attacks before damage is done. Modern authentication
protocols like OAuth 2.0 are designed to offload security tasks to third-party providers. When
additional safeguards are built into the system, they often incur significant infrastructure complexity
and performance costs.

<!-- truncate -->

## OAuth 2.0

[OAuth 2.0] is a widely used protocol for managing access to resources on the web.
In this system, there are two types of server entities:

1. An **authorization server** that logs in users and issues access tokens.
2. A **resource server** that has the private data the user is trying to access, such as their
   photos, videos, or emails.

First, the user presents credentials to an authorization server. For example, the user may be redirected
to a login form where they need to enter their email address, password, and/or two-factor authentication code.
In exchange for a valid set of credentials, the authorization server issues tokens for making API requests
to the resource server. Most implementations return tokens in [JWT] format which consist of a header,
JSON-encoded user attributes, and cryptographic signature to prevent tampering.

![OAuth 2.0 Simplified Block Diagram](/img/oauth2.svg)

In a simple system, the authorization server and resource server may be different groups of API endpoints
residing on the same backend application. However, separating out the authorization server has many critical
benefits. Resource servers don't need to handle passwords and other sensitive user data. All they need to do is
validate the token and read out the user IDs inside. Since tokens typically use RSA signatures,
the process simply requires a copy of the authorization server's public key.

The separation of the authorization server also allows us to delegate user management to a third-party.
For example, a website can implement a "Sign in with Google" feature, so users don't need to register a separate
username and password with them. Or, the authorization server can be provided by a cloud service, such as Okta
or AWS Cognito.

## Access Tokens and Refresh Tokens

An authorization server may return an access token and a separate refresh token. The access token is used
to make requests to the resource server. It is valid for a short period of time, usually 1 hour.
The expiration date is stored in the token payload and checked whenever the token is decoded.

```javascript
// A sample access token payload
{
   "jti": "073a91dc-0b34-4d7e-8f21-a68aff6dae57",  // token ID
   "sub": "user_12345",                            // subject ID (user ID)
   "iss": "https://auth.example.com",              // token issuer
   "aud": "266858dc5f64",                          // audience (client ID)
   "iat": 1680103887,                              // issued-at time
   "exp": 1680107487,                              // expiration time
}
```

The refresh token is used to obtain a new access token from the authorization server when the current one
expires. By increasing the lifetime of the refresh token, a website can implement a "remember me" feature,
where the user remains logged in across browser sessions.

## Vulnerabilities

OAuth clients must protect access tokens and refresh tokens from being leaked to an adversary.
A stolen access token can be used to impersonate a user and obtain their private data from the resource server.
A stolen refresh token can be used to obtain fresh access tokens from the authorization server,
allowing the attacker to maintain their presence for an extended period of time.

Tokens are protected in transit by enforcing TLS connections. Communication with the authorization server
may be protected with CSRF tokens, nonces, and URL comparisons to prevent inadvertent redirects to a malicious site.

Protecting tokens at rest is a harder problem. Web browsers must save authentication state on the filesystem
so that users can resume a previous browsing session, undo a closed tab, or even recover from a crash.
A malicious application could read these files and extract the tokens. Antivirus programs might be able
to flag unusual file accesses, or we could prevent them through the use of sandboxing.
However, these techniques are not completely reliable and may not be supported on all environments.

![OAuth 2.0 Token Attack](/img/oauth2_attack.svg)

## Server-Side Defenses

### Location Analysis

In a typical scenario, an adversary exfiltrates access tokens from the target's computer, then uses those tokens
to perform API requests from their own computer. To prevent this attack, the resource server can reject
requests coming from an IP address or location that does not match the one to which the token was originally issued.

Restricting an API by location is a non-trivial problem. We cannot assume that IP addresses are static and bound
to a single person. Some internet providers assign IPs dynamically from a pool of addresses. There could be NATs or proxies
where multiple users share a single address. Users may travel and connect to different network access points.

To avoid locking out a legitimate user, our resource server could track connections at a less granular level,
focusing on IP subnets or approximate geolocation. It may use additional context, such as the rate of API requests and time of day,
to arrive at a risk score for every incoming connection. For a user based in Los Angeles, a single API request from
New York City is suspicious, but a sequence of API requests from a different IP address in Los Angeles might be normal.

![OAuth 2.0 Location Detection](/img/oauth2_geo.svg)

However, location analysis imposes a significant infrastructure burden on our system. The core principle of OAuth 2.0 is
that a vast number of resource servers can delegate complex authentication tasks to an external provider.
We don't want to make each resource server maintain its own IP geolocation database. But what happens if these tasks are
done on an authorization server? Before the resource server accepts an access token, it would have to make a blocking call
to the central server to confirm the token is still valid for that origin. The backchannel between the authorization
server and the resource server becomes a potential bottleneck, requiring caching and DDoS defenses. Coordinating with
cloud providers on a standard implementation might not be possible.

![OAuth 2.0 Location Infrastructure](/img/oauth2_geo2.svg)

### Token Revocation

If we do detect suspicious activity, we need to revoke the access token and refresh token, so the user
will be forced to log in again with their credentials. We should also add a "sign out all other sessions"
button to our application so users can lock down their accounts manually.

Authorization servers can maintain a revocation list of token IDs that must never be accepted, regardless
of expiration time. Resource servers need to query this list periodically to validate incoming
access tokens.

But once again, we have to consider the complexities of distributed systems. To protect
the authorization server from traffic spikes, we may need to cache the revocation list. Changes to the
revocation list may take time to propagate to all resource servers. Token revocation is at best
_eventually consistent_, and may not occur quickly enough to stop every fraudulent API call.

![OAuth 2.0 Token Revocation](/img/oauth2_rev.svg)

### Access Token Scopes

Meanwhile, to limit the damage that can be done with a compromised access token, we should bind tokens
to specific API scopes. By default, an access token should not grant permission to reset a password,
change the email address associated with an account, or perform other administrative actions.
Access to sensitive API endpoints should require the user to reauthenticate and receive a special
access token. These tokens can have shorter expiration times or be valid only for a single use.

```javascript
// A sample access token payload, with scope attribute
{
   "jti": "073a91dc-0b34-4d7e-8f21-a68aff6dae57",
   "sub": "user_12345",
   "iss": "https://auth.example.com",
   "aud": "266858dc5f64",
   "iat": 1680103887,
   "exp": 1680107487,

   // Authentication scopes in OpenID Connect, an OAuth 2.0 extension.
   // Implementations can add custom values to limit tokens to
   // specific API permission levels.
   "scope": "openid profile email",
}
```

## Security Tradeoffs

Detecting the misuse of an OAuth 2.0 token adds complexity to an already elaborate protocol.
It introduces dependencies between the authorization server and the resource servers.
Often, there are impacts on user convenience, API latency, site reliability, and financial cost.

In a high-risk environment, the drawbacks are acceptable. Users of a banking website will probably
tolerate being logged out after 10 minutes of inactivity, or being forced to reauthenticate after
connecting from a new location. However, other websites may decide to err on the side of infrastructure
simplicity and speed. Official standards on authentication do not enumerate all possible countermeasures
against token misuse, leaving them as implementation details and best practices for the experienced developer.
As a result, it is not surprising that security breaches continue to occur.


[recent attack]: https://youtu.be/yGXaAWbzl5A
[OAuth 2.0]: https://www.rfc-editor.org/rfc/rfc6749
[JWT]: https://www.rfc-editor.org/rfc/rfc7519
