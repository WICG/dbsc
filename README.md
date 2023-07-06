# Device Bound Secure Credentials explainer

This is the repository for Device Bound Secure Credentials. You're welcome to
[contribute](CONTRIBUTING.md)!

## Authors:
- [Kristian Monsen](kristianm@google.com), Google
- [Arnar Birgisson](arnarb@google.com), Google

## Participate (to come)
- [Issue tracker]
- [Discussion forum]

## Table of Contents [if the explainer is longer than one printed page]
<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
  - [Goals [or Motivating Use Cases, or Scenarios]](#goals-or-motivating-use-cases-or-scenarios)
  - [Non-goals](#non-goals)
  - [What makes Device Bound Secure Credentials different](#what-makes-device-bound-secure-credentials-different)
    - [Application-level binding](#application-level-binding)
    - [Browser-initiated refreshes](#browser-initiated-refreshes)
  - [TPM considerations](#tpm-considerations)
  - [Privacy considerations](#privacy-considerations)
- [High level overview](#high-level-overview)
  - [Start Session](#start-session)
  - [Maintaining a session](#maintaining-a-session)
    - [Refresh procedure](#refresh-procedure)
    - [Ending a session](#ending-a-session)
- [Interactions with other APIs](#interactions-with-other-apis)
  - [Login Status API](#login-status-api)
  - [Interaction with Inactive Documents (BFCache, Prerendering)](#interaction-with-inactive-documents-bfcache-prerendering)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Introduction
Device Bound Secure Credentials (DBSC) aims to reduce account hijacking caused by cookie theft. It does so by introducing a protocol and browser infrastructure to maintain and prove possession of a cryptographic key. The main challenge with cookies as an authentication mechanism is that they only lend themselves to bearer-token schemes. On desktop operating systems, application isolation is lacking and local malware can generally access anything that the browser itself can, and the browser must be able to access cookies. On the other hand, authentication with a private key allows for the use of system-level protection against key exfiltration.

DBSC offers an API for websites to control the lifetime of such keys, behind the abstraction of a session, and a protocol for periodically and automatically proving possession of those keys to the website's servers. One of the key goals is to enable drop-in integration with common types of current auth infrastructure. By device-binding the private key and with appropriate intervals of the proofs, the browser can limit malware's ability to offload its abuse off of the user's device, significantly increasing the chance that either the browser or server can detect and mitigate cookie theft.

DBSC is bound to a device with cryptographic keys that cannot be exported from the user’s device under normal circumstances, this is called device binding in the rest of this document. DBSC provides an API that servers can use to create a session bound to a device, and this session can periodically be refreshed with an optional cryptographic proof the session is still bound to the original device. At sign-in, the API informs the browser that a session starts, which triggers the key creation. It then instructs the browser that any time a request is made while that session is active, the browser should ensure the presence of certain cookies. If these cookies are not present, DBSC will hold network requests while querying the configured endpoint for updated cookies.

### Goals
Reduce session theft by offering an alternative to long-lived cookie bearer tokens, that allows session authentication that is bound to the user's device. This makes the internet safer for users in that it is less likely their identity is abused, since malware is forced to act locally and thus becomes easier to detect and mitigate. At the same time the goal is to disrupt the cookie theft ecosystem and force it to adapt to new protections.

### Non-goals
DBSC will not prevent temporary access to the browser session while the attacker is resident on the user’s device. The private key should be stored as safe as modern desktop operating systems allow, preventing exfiltrating of the session private key, but the signing capability will still be available for any program running as the user on the user’s device. 

### What makes Device Bound Secure Credentials different
DBSC is not the first proposal towards these goals, with a notable one being [Token Binding](https://en.wikipedia.org/wiki/Token_Binding). This proposal offers two important features that we believe makes it easier to deploy than previous proposals. DBSC provides application-level binding and browser initiated refreshes that can make sure devices are still bound to the original device.
#### Application-level binding
For websites, device binding is most useful for securing authenticated sessions of users. DBSC allows websites to closely couple the setup of bound sessions with user sign-in mechanisms, makes session and key lifetimes explicit and controllable, and allows servers to design infrastructure that places verification of session credentials close to where user credentials (cookies) are processed in their infrastructure.

Alternatives such as Token Binding gain much from e.g. integrating with TLS, but this can make integration harder in environments where e.g. TLS channel termination is far removed from the application logic behind user sign-in and session management.
#### Browser-initiated refreshes
Other proposals have explored lower-level APIs for websites to create and use protected private keys, e.g. via WebCrypto or APIs similar to WebAuthn. While this works in theory, it puts a very large burden on the website to integrate with. In particular, since the cost of using protected keys is high, websites must design some infrastructure for collecting signatures only as often as needed.

This means either high-touch integrations where the keys are only used to protect sensitive operations (like making a purchase), or a general ability to divert arbitrary requests to some endpoint that collects and verifies a signature and then retries the original request. The former doesn't protect the whole session and violates the principle of secure by default, while the latter can be prohibitively expensive for large websites built from multiple components by multiple teams, and may require non-trivial rewrites of web and RPC frameworks.

DBSC instead allows a website to consolidate the session binding to a few points: At sign-in, it informs the browser that a session starts, which triggers the key creation. It then instructs the browser that any time a request is made while that session is active, the browser should ensure the presence of certain cookies. The browser does this by calling a dedicated refresh endpoint (specified by the website) whenever such cookies are needed, presenting that endpoint with a proof of possession of the private key. That endpoint in turn, using existing standard Set-Cookie headers, provides the browser with short-term cookies needed to make other requests.

This provides two important benefits:

1. Session binding logic is consolidated in the sign-in mechanism, and the new dedicated refresh endpoint. All other parts of the website continue to see cookies as their only authentication credentials, the only difference is that those cookies are short-lived. This allows deployment on complex existing setups, often with no changes to non-auth related endpoints.

1. If a browser is about to make a request where it has been instructed to include such a cookie, but doesn't have one, it defers making that request until the refresh is done. While this may add latency to such cases, it also means non-auth endpoints do not need to tolerate unauthenticated requests or respond with any kind of retry logic or redirects. This again allows deployment with minimal changes to existing endpoints.

Note that the latency introduced by deferring of requests can be mitigated by the browser in other ways, which we discuss later.

### TPM considerations
DBSC depends on user devices having a way of signing challenges while protecting private keys from exfiltration by malware. This usually means the browser needs to have access to a Trusted Platform Module (TPM) on the device, which is not always available. TPMs also have a reputation for having high latency and not being dependable. Having a TPM is a requirement for installing Windows 11, and can be available on previous versions.

 Chrome has done studies to understand TPM availability to understand the feasibility of secure sessions. Current data shows more than 50%, and currently growing, of Windows users would be offered protections. Studies have also been done on the current populations of TPMs, both for latency and for predictability. Currently the average is around 0.5 seconds for signing operations, but with large variations. There are also significant errors in operations, but well below 1%.

Based on this research, TPMs are widely available, with a latency and consistency that is acceptable for the proposed usage.
### Privacy considerations
An important high-level goal of this protocol is to introduce no additional surface for user tracking: implementing this API (for a browser) or enabling it (for a website) should not entail any significant user privacy tradeoffs.

There are a few obvious considerations to ensure we achieve that goal:
- Lifetime of a session/key material: This should provide no additional client data storage (i.e., a pseudo-cookie). As such, we require that browsers MUST clear sessions and keys when clearing other site data (like cookies).
- Cross-site/cross-origin data leakage: It should be impossible for a site to use this API to circumvent the same origin policy, 3P cookie policies, etc. (More on this below.)
- Implementing this API should not meaningfully increase the entropy of heuristic device fingerprinting signals. (For example, it should not leak any stable TPM-based device identifier.)
- This API—which allows background "pings" to the refresh endpoint when the user is not directly active—must not enable long-term tracking of a user when they have navigated away from the connected site.

## High level overview
The general flow of a secure session is as follows:
1. The website requests that the browser start a new session, providing an HTTP endpoint to negotiate registration parameters.
1. The browser creates a device-bound key pair, and calls the registration HTTP endpoint to set up the session and register the public key.
1. The server responds with a session identifier, and instructions on how to maintain the session. This is a (possibly empty) list of cookie names that the browser is expected to ensure exist, and their associated scope (origins+path).
1. At a later time the session is closed by either the server requesting to close the session or the user clears the keys by clearing site data.

As long as that session is active, the browser performs the following refresh as needed:
1. For any request within an applicable scope, the browser checks if the necessary cookies exist. If they exist, continue as normal.
1. If not, the browser defers such requests while performing the following.
1. The browser contacts the HTTP endpoint for the session, providing the session identifier and, if requested by the server, the necessary proof of possession of the associated private key. 
1. If the server is satisfied with the proof, it uses regular Set-Cookie headers to establish the necessary cookies with an appropriate Max-Age. The response can also include an updated set of instructions, e.g. if the server wishes to change which cookies are subject to this logic.
1. If any requests were deferred in step 2, the browser now makes those requests including the updated set of cookies.
1. The browser may choose to proactively refresh cookies that are about to expire, if it predicts the user may soon need them. This is purely a latency optimization, and not required.

### Start Session
![Start session diagram](start.png)

The API consists of a new interface, SecureSession, an instance of which is obtained via the securesession property of the navigator object. The SecureSession interface supports the following method:

- **startSession():**
- Parameters:
  - endpoint: The URL of the secure session endpoint, supporting /startsession and /refresh operations
  - supported_binding_algs: Array of cryptographic algorithms supported by the server
  - authorization: Optional authorization code to be passed to Start Session HTTP request
- Returns:
  - A promise with a string containing the session ID (which may be empty)

When called, this method must:
- Generate and store a new, secure cryptographic key pair
- Call the "endpoint" + /startsession as specified below in Start Session (HTTP Request/Response)
- Return a promise which
  - If that call succeeds (i.e. returns an HTTP 200), returns a session ID obtained from that call (which can be the empty string)
  - If that call fails, throws an exception (which may indicate the HTTP status of the failed call)

Requirements:
- The endpoint must have the same origin as the JavaScript.

Below is an example of the API being used:
```javascript
let promise = navigator.securesession.start({
  // Session start options
  "endpoint": "<url prefix of standard session endpoint>", // required
  "supported_binding_algs": ["ES256,RS256"], // required
  "authorization": "<authorization code>", // optional
});
promise.then((sessionInfo) => {
  // Success means the browser has completed the session setup with the
  // session endpoint and will perform the necessary maintenance tasks
  // going forward.
  console.log("Session with id {} was started.", sessionInfo.id);
});
promise.catch((...) => {
  // Session start failed for some reason, e.g. the HTTP endpoint was
  // not reachable, or broke protocol.
  <error handling>
});
```

The browser responds to the session start by selecting a compatible signature algorithm and creating a device-bound private key for the new session. It then makes the following HTTP request (assuming the endpoint URL is https://auth.example.com/securesession):

```http
POST /securesession/startsession HTTP/1.1
Host: auth.example.com
Accept: application/json
Content-Type: application/json
Content-Length: nn
Cookie: whatever_cookies_apply_to_this_request=value;
```
```json
{
  "binding_alg": "ES256",
  "binding_public_key": <base64url encoded generated public key>,
  "client_constraints": {
    "signature_quota_per_minute": 2,
  },
}
```

If the request is properly authorized, the server establishes whatever state represents the session server-side, and returns the following response.

```http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store
Set-Cookie: auth_cookie=abcdef0123; \
            Domain=example.com; Max-Age=600; Secure; HttpOnly;
```
```json
{
  "session_identifier": "<server issued identifier for the session>",
  "credentials": [{
    "name": "auth_cookie",
    "excluded_scope": [
      // the path is a prefix
      "path_a",
      "path_b"
    ]
  }],
}
```

If the browser accepts this response, it successfully resolves the promise from the initial JS call with the session identifier (this allows for some decoupling of sign-in and session-management services on the server side).

Subsequently, as long as the browser considers this session "active", it follows the steps above, namely by refreshing the auth_cookie whenever needed, as covered in the next section.

### Maintaining a session
As long as the named cookie is not expired the browser will keep sending requests as normal. Once the cookie is expired the browser will hold all requests for the scope of the cookie, except where the server excluded the paths in the registration, while refreshing the cookie. This is where the browser driven protocol makes a difference, if not for this there would be potentially many requests without the required cookie.
#### Refresh procedure
![Refresh diagram](refresh.png)

The browser refreshes the short-term session credential by calling the session endpoint:

```http
GET /securesession/refresh HTTP/1.1
Host: auth.example.com
Content-Type: application/json
Content-Length: nn
Cookie: whatever_cookies_apply_to_this_request=value;
Sec-Session-Id: [session ID]
```

In response to this the server can optionally first request a proof of possession of the key by issuing a challenge to the browser by responding with a 401 response with a challenge:

```http
HTTP/1.1 401
Sec-Session-Challenge: session_identifier=<session identifier>,challenge=<base64url encoded challenge>
```

The browser replies to that response with a Sec-Session-Response header, containing a signed JWT:

```http
GET /securesession/refresh HTTP/1.1
Sec-Session-Response: <base64-URL-encoded JWT>
```

The JWT contains:
```json
{
  "jti": "challenge from Sec-Session-Challenge header",
  "aud": "the URL to which the Sec-Session-Response will be sent",
  "sub": "the session ID corresponding to the binding key",
}
```

If the server is satisfied with the response, or if it did not request it, it answers by setting the short term cookie. Optionally the server can adjust the session in the body of the response similar to how it was set up.

```http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store
Set-Cookie: auth_cookie=abcdef0123; \
             Domain=example.com; Max-Age=600; Secure; HttpOnly;
```
With the following contents:
```json
{ // optionally adjusting the session
  // id must be the same as previously
  "session_identifier": "<server issued identifier for the session>",
  "credentials": [{
    "name": "auth_cookie",
    "excluded_scope": [
      // the path is a prefix
      "path_a",
      "path_b"
    ]
  }],
}
```
On receiving this response, the browser releases any requests that were deferred pending this refresh, including the new cookie.
Note:
This response is nearly identical to the response to session setup, and can be handled by the same logic in the client.
The new set of instructions replaces any previous instructions for the session. E.g. if the cookie name is different here than before, the browser should not trigger refreshes based on the absence of the old cookie name.
The server may issue a different Max-Age, or scope for the short-term cookie.
The server may set or update other cookies not subject to session refreshes in this response, as in any response.

If instead the server decides to end the session it can respond with:
```json
{
  "session_identifier": "<server issued identifier for the session>",
  "continue": false
 }
```
In this case the browser should stop triggering any refreshes for this request, release any deferred requests without the short-term cookie, and clean up and delete the associated session state including the binding key.

#### Ending a session
The session can end in several ways:
- The server can end the session during a refresh by answering with {“continue”: false} during a refresh
- The server can at any time send a header with Clear-Site-Data: "storage"
- The user can clear site data, which will locally clear cookies and any registered session keys for the site

It is important that the user is always in control and can delete the session keys if wanted.

## Interactions with other APIs
### Login Status API
### Interaction with Inactive Documents (BFCache, Prerendering)
When a session is ended for any reason, any inactive documents which had access to that session's credentials should be destroyed. This ensures that pages in BFCache or that are pre-rendering that contain information guarded by those credentials are not presented after the session has ended.