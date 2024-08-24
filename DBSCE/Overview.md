<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Device Bound Session Credentials for Enterprise - explainer](#device-bound-session-credentials-for-enterprise---explainer)
  - [Authors](#authors)
  - [Contributors](#contributors)
  - [Participate (TBD links)](#participate-tbd-links)
  - [Overview](#overview)
  - [Why DBSC(E)?](#why-dbsce)
  - [How does it integrate with DBSC?](#how-does-it-integrate-with-dbsc)
  - [High-Level Design](#high-level-design)
  - [Terminology](#terminology)
    - [Browser](#browser)
    - [Relying Party (RP)](#relying-party-rp)
    - [Identity Provider (IdP)](#identity-provider-idp)
    - [Device Registration Client](#device-registration-client)
    - [Local Key Helper](#local-key-helper)
      - [Platform Examples:](#platform-examples)
    - [Attestation Service:](#attestation-service)
    - [Device Registration (Pre-Session)](#device-registration-pre-session)
    - [DBSC(E) use cases](#dbsce-use-cases)
      - [IDP Calls Public Local Key Helper](#idp-calls-public-local-key-helper)
      - [IDP is RP and Calls Public Local Key Helper](#idp-is-rp-and-calls-public-local-key-helper)
      - [IDP Calls Private Local Key Helper](#idp-calls-private-local-key-helper)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Device Bound Session Credentials for Enterprise - explainer

This is the repository for Device Bound Session Credentials for Enterprise. You're welcome to
[contribute](CONTRIBUTING.md)!

## Authors

- [Sameera Gajjarapu](sameera.gajjarapu@microsoft.com), Microsoft
- [Aleksander Tokarev](alextok@microsoft.com), Microsoft

## Contributors

- [Olga Dalton](), Microsoft
- [Kristian Monsen](), Google
- [Phil Leblanc](), Google
- [Sebastian](), Google
- [Arnar Birgisson](), Google
- [Pamela Dingle](), Microsoft
- [Paul Garner](), Microsoft
- [Erik Anderson](), Microsoft
- [Will Bartlett](), Microsoft
- [Kai Song](), Microsoft
- [Amit Gusain](), Microsoft

## Participate (TBD links)

- [Issue tracker]()
- [Discussion forum]

## Overview

Device Bound Session Credentials for Enterprise - DBSC(E), is an enhancement to the existing [DBSC](https://github.com/wicg/dbsc) proposal. It refines the key generation mechanism resulting in additional security for enterprise use cases. It aims to provide a mechanism for enterprise and advanced browser customers to be able to deploy enhanced/customised device binding for any browser session, hence protecting against session hijacking and cookie theft.

## Why DBSC(E)?

While the original DBSC proposal is focused on providing a mechanism for browsers to bind session credentials to a device, it still remains vulnerable to malware that can run on a device during any web application signin/login. If a malware happens to be already running in the device, it can force the user to login, and provide its own binding keys (asymmetric key pair) to the web application, there by gaining the ability to steal the session. Any upcoming sessions after this, even with DBSC, will not be reliable.

DBSC(E) aims to mitigate this risk by introducing the concept of key generation (asymmetric device-bound key) during the device registration and binds all the future sessions to the key. Enterprises can decide when and how to generate device bound keys, however, DBSC(E) allows for a given session to be bound to the device, if the device registration is performed when there is no malware on the device (a state referred to as ["clean room"](#device-registration-client)) e.g. before giving a device to an employee. Device registration is also expected to be a once-in-a-lifetime protected operation, and hence the user will not be required to perform this operation again, reducing opportunities for malware to extract session credentials of the device.DBSC(E) makes it impossible for malware to compromise a device during signin/login, but DBSC(E) doesn't protect a given session if the malware is present during the device registration.

## How does it integrate with DBSC?

DBSC(E) is not intended to be a separate proposal from DBSC, it is rather building on existing DBSC, and adds the binding specific details to the protocol. It is expected that the DBSC(E) proposal will be integrated into the DBSC proposal in the specification. In the high-level design, we have folded the DBSC proposal into the end to end flow. Please read the [DBSC proposal](https://githuub.com/wicg/dbsc) before you proceed.

Before we get into the specifics, we will introduce the terminology and design specifics for the key generation and validation below.

## Terminology

### Browser

In this document, "Browser" refers to the functionality in a web browser that is responsible for the DBSC protocol. This functionality will be implemented by Edge, Chrome (or their common engine), and other browsers that choose to implement DBSC/DBSC(E).

### Relying Party (RP)

A web application that uses DBSC(E) protocol for cookie binding. This is referred to as `server` in the original [DBSC design](https://githuub.com/wicg/dbsc).

### Identity Provider (IdP)

IdP is an authentication server that can be either external to the Relying Party or part of the Relying Party. Eg: Office.com authenticating with Microsoft Entra ID or google.com authenticating with google. Note: The protocol doesn't change if the IdP is part of the Relying Party, except that some redirects between the IdP and the RP can be skipped or implemented by other means. IDP and RP are same for the certain consumer/enterprise use cases and is referred to as `server` in the original [DBSC design](https://githuub.com/wicg/dbsc).

### Device Registration Client

This is a pre-requisite for DBSC(E) to work.

Device Registration Client is a process where the user or administrator registers the device with the IdP and is expected to be a once-in-a-lifetime protected operation.

The device registration establishes trust between the device and a service that maintains a directory of all devices. This document does not cover the protocol of device registration, but it assumes that during device registration, some asymmetric keys are shared between the client and the service, typically a device key and some other keys necessary for the secure device communication. A client software component that performs the device registration is called a _device registration client_. As mentioned above, the key assumption in DBSC(E) is that device registration happened in a clean room environment, and it is the responsibility of the device owner to ensure this.

One device registration client can manage multiple devices on the same physical device. There also can be multiple device registration clients on the same device. The device registration client can be owned and supported by:

- Operating system - the device gets registered when the OS is installed.
- Browser - the device gets registered when the browser is installed.
- Device management software (MDM provider) - the device gets registered when the MDM is enrolled.
- Third-party software vendor - the device gets registered according to the vendor rules.

DBSC(E) aims to support most of these scenarios. It does not define the device registration protocol and is only concerned with the keys generated in a "clean room" state and the management of the generated keys to prove device binding.


### Local Key Helper

DBSC(E) introduces the concept of `Local Key Helper`.

**Local Key Helper** is an integral part of the the **Device Registration Client**, a software interface responsible for the DBSC Key management. *Local key helper* can be public or private and is expected to be either shipped as a part of a given enterprise framework (with the IdP/OS) or can be installed by a provider in compliance with the protocol expanded below.

From the deployment point of view there are two types of local key helpers: _well-known_(_private_) and _third party_(_public_)

- _Public local key helper_ or _third party_: Can be accessed by any Identity Provider (IdP). Typically owned by a provider different from the IdP, communicates with the IdP through as defined in DBSC(E) protocol. A third party local key helper has a special deployment mechanism. We expect to loosely define the API and deployment mechanism in the upcoming specification, and will leave the specific implementation details to the vendors. However, we have provided examples of how this can look for a few platforms [below](#platform-examples).
- _Private local key helper_ or _well known_ : Can be specific to an IdP. Typically owned by the IdP and will have a private protocol to communicate with the IdP. Comes with either OS or built into the browser. It assumed that well-known key helpers are **trusted and enabled by default** in a browser and/or a given IDP. A browser knows how to activate a well-known key helper.

The Local Key Helper is responsible for:

- Generation of the binding key and producing binding statements (see below)
- Producing signatures with the binding key
- Cleanup of the binding key and its artifacts (when the user clears the browser session or the key is unused for a long time).

#### Platform Examples

- Windows: Please refer to the Windows Local Key Helper [here](./KeyGeneration.md#local-key-helper-on-windows) for an example of a Local Key Helper.
- MacOS: TBD
- Android:TBD

Note: Above are examples of Local Key Helpers that can be used for DBSC(E) key generation. Any platform is free to build a version of this, as long as the API is consistent with the protocol.

### Attestation Service:

A service that is responsible for verifying that the binding key is issued by the expected device and providing the attestation of the key to the IdP. The attestation service can be owned by the IdP or a third party. DBSC(E) relies on the attestation service to validate the binding statement and ensure that the binding key and the device key belong to the same device. We have added details on the specifics of the binding artifacts generated during the device registration process, and the validation of the binding statement in the [DBSC(E) Key Generation](#key-generation-specifics) section.

### Device Registration (Pre-Session)

Any enterprise user is expected to either use a device issued by their organization or register their personal device with the organization. The device registration is expected to be a once-in-a-lifetime operation, and the user is expected to perform this operation in a clean room environment.

![DeviceRegistration](./DeviceRegistration.svg)

## High-Level Design

DBSC(E), if enabled for a given enteprise, specifies the generation of the cryptographic artifacts (keys and binding info) before a sign in session is established. By enabling the browser to invoke specific APIs based on an existing policy, it allows enterprises to add to the existing key generation. It also allows them to place stricter restrictions on specific sessions, hence providing the flexibility to secure a session appropriately.

The high-level design is divided into two parts:

1. Key generation and validation before the session starts (DBSC(E) is focused on this part).
2. DBSC protocol applied with the generated keys (DBSC is focused on this part).

Since we want to integrate DBSC(E) with the original design and make it as widely applicable as possible for all enterprise users, we are adding high-level design for the most possible combinations in this document. The intent is to have a specification that can be implemented by any browser vendor, and can be used by any IdP, and any Local Key Helper. As we cover different use cases DBSC(E) can be applied for, we differentiate between private and public local key helpers, since there are implications to the protocol based on the type of local key helper. For example, we expect well establised IdPs like Microsoft, Okta, Github to ship their own local key helper as a part of the IdP. We also optimize the DBSC(E) protocol for RP and IdP as the same service (google client authenticated by google service as an example), since it simplifies the protocol and aligns with the perf goals for specific services.

DBSC(E) (in contrast with DBSC):

![DBSC(E) Highlevel Design](<./DBSC(E).svg>)

Highlights:

Note: All references to RP, IDP are equivalent to `server` in the original [DBSC design](https://githuub.com/wicg/dbsc).

1. **Pre-Session initiation with special headers (steps 1-2):** When a user starts a sign-in process, or initiates a session, the webpage initiating the session sends special headers `Sec-Session-GenerateKey` and `Sec-Session-HelperIdList` to the browser in response, to indicate that the session is expected to be DBSC(E) compliant.

   - The `Sec-Session-GenerateKey` header contains the URL of the server(RP), the URL of the IdP (authentication service in most cases - which is optional for consumer use cases), a `nonce`(challenge) and any extra parameters that the IdP wants to send to the Local Key Helper.
   - As the Local Key Helper is public or thirdparty, and potentially can be from a different vendor, `nonce`(challenge) is essential to prevent the clock-skew between the IdP and the Local Key Helper, especially for short lived Binding statements. `nonce`(challenge) is expected to be generated by the IdP/RP as a part of the request, a random number with time sensitivity, and is expected to be embedded in the binding statement. IdP will be able to validate nonce to ensure the binding statement is freshly issued.
   - The `Sec-Session-HelperIdList` header contains a list of helper IDs that the browser can use to generate the key. As we touched upon before, there could be multiple devices on a single physical device, and/or multiple device registration clients on a single device. The `HelperId` helps the browser to choose the right **Local Key Helper** to generate the key. The browser will evaluate the policy for the IdP and the helper IDs, and choose the appropriate helper ID to generate the key. The browser will then call the Local Key Helper to generate the key.

1. **Key and Binding Statement Generation (steps 3-7):** The Local Key Helper generates the key and the binding statement. AIK refers to the `Attestation Key` described [above](#attestation-key). The binding statement is expected to contain the `nonce`(challenge) sent by the IdP, the thumbprint of the public key, and any extra claims that the IdP wants to send.

   - Format of the Binding Statement: We expect the `binding statement` will be a `string`, as we want to keep the format open to allow for platform-specific optimizations. However, the validation of the `binding statement` is prescribed to include `nonce`(challenge) and the thumbprint of the public key. The `binding statement` is expected to be shortlived to prevent forgery of the binding statement from a different device. More details on `binding statement` can be found [here](./KeyGeneration.md#binding-statement).
   - The `extra claims` is a provision added for specific IdPs or Local Key Helper vendors to add any additional information to the `binding statement`. It is intentionally left undefined, and can be customized.
   - Local Key Helper can optionally signal the browser to cache the Binding Statement in the browser. This is to avoid repeated calls to the Local Key Helper for the same IdP. The browser can cache the Binding Statement for a certain time, and if the IdP requests a new key within that time, the browser can return the cached Binding Statement. TBD define under which conditions.

1. **Sign In/Session Initiation (steps 8-9):** The `Binding Statement`, with the `KeyId` is expected to be returned to the IdP witha new header, `Sec-Session-Keys`. The IdP will [validate](#binding-statement---generation-and-validation) the signature on the `Binding Statement`, `nonce` and stores the thumbprint of the public key. Once the validation succeeds, the IdP will proceed with the sign-in ceremony (optionally generate auth tokens if the RP and IdP are separate, illustrated [below](#idp-is-rp-and-calls-public-local-key-helper), that embed the thumbprint of the public key). The `KeyId` is expected to be returned to the RP/IdP, and the IdP will use the `KeyId` to identify the key to be used for the session.

1. **SignIn Succeeds with binding (steps 10-14)**: At this point, all DBSC(E) specific steps are completed. The server returns signed in content with a special header response to the browser: `Sec-Session-Registration` indicated the session is expected to be DBSC compliant. All steps further are as per the original DBSC proposal with additional params introduced for DBSC(E) customization.

### DBSC(E) use cases

This section expands on the [generic design](#high-level-design) to address different enterprise use cases:

#### IDP is RP and Calls Public Local Key Helper

This is the same use case elaborated in the high level design [above](#high-level-design). However, we have separated the IdP/RP components of the server for the enterprise use case in the below diagram.

![IDPSameAsRP-CallsPublicLocalKeyHelper](./IDPSameAsRP-CallsPublicLocalKeyHelper.svg)

Highlights:

- Auth tokens are not mentioned in the DBSC(E) high level design, as they are optional for the consumer use case. However, for the enterprise use case, the IdP can generate the auth tokens and embed the thumbprint of the public key in the token.
- The token can be delivered to the RP directly through an API instead of a header based response.
- The tokens also contain the thumpbrint of the public key, and the RP can validate the thumbprint of the public key from the token.

#### IDP Calls Public Local Key Helper

In Enterprise, and some consumer use cases, it is a valid use case to have separate servers as [RP](#relying-party-rp) and [IdP](#identity-provider-idp). We address the use case where these are separate entities and probably from different vendors below.

For easy mapping with the existing DBSC proposal, please note:

- Steps 1-16 specify the key generation process for a public local key helper.
- Steps 17-29 are [DBSC](https://github.com/wicg/dbsc), added for completeness.

![IDPCallsPublicLocalKeyHelper](./IDPCallsPublicLocalKeyHelper.svg)

#### IDP Calls Private Local Key Helper

A special case is for enterprises that already have `well-known` Local Key Helpers, which are expected to be trusted and enabled by default in a browser. Here, since the browser can trust a given IDP (based on the URL and a policy mapping) and can trust the IDP to invoke the appropriate Local Key Helper (refer [Local key helper on Windows](./KeyGeneration.md#local-key-helper-on-windows)), there are a few optimizations that can be made to the protocol, in skipping the `nonce`(challenge) and reducing the number of round trips between the IDP and the Local Key Helper.

![IDPCallsPrivateLocalKeyHelper](./IDPCallsPrivateLocalKeyHelper.svg)

Highlights:

- Since the IDP (a specific URL) is trusted in this use case, the browser can skip the `nonce`(challenge) and directly call the Local Key Helper.
- Attestation Service is optional as it is assumed that the IDP is trusted and can validate the binding statement. In other words, Attestation Service is assumed to be part of the IDP in this case.
- The headers are modeled after confidential client flow(OAuth2) implemented with [Microsoft IDP](https://learn.microsoft.com/en-us/entra/identity-platform/refresh-tokens) as an example. It is possible to replace `Microsoft` with `Okta` and achieve the same flow.

### Cleanup of the binding keys and their artifacts

For the health of the operating system and user privacy, it is important to clean up binding keys and their artifacts. If the number of keys grows too high, the performance of the OS can degrade.

The cleanup can occur:

- On demand, when the user decides to clear browser cookies.
- Automatically, when a key hasn't been used for a long time (N days) to keep the OS healthy.
