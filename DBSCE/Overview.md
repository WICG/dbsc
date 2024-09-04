<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Device Bound Session Credentials for Enterprise - explainer](#device-bound-session-credentials-for-enterprise---explainer)
  - [Authors](#authors)
  - [Contributors](#contributors)
  - [Participate (TBD links)](#participate-tbd-links)
  - [Overview](#overview)
  - [Why DBSC(E)?](#why-dbsce)
  - [How does it integrate with DBSC?](#how-does-it-integrate-with-dbsc)
  - [Terminology](#terminology)
    - [Browser](#browser)
    - [Relying Party (RP)](#relying-party-rp)
    - [Identity Provider (IdP)](#identity-provider-idp)
    - [Device Registration Client](#device-registration-client)
    - [Device Registration](#device-registration)
    - [Local Key Helper](#local-key-helper)
      - [Platform Requirements](#platform-requirements)
    - [Attestation Service](#attestation-service)
      - [Key Generation Specifics](#key-generation-specifics)
        - [Binding Key](#binding-key)
        - [Attestation Key](#attestation-key)
        - [Binding Statement](#binding-statement)
  - [High-Level Design](#high-level-design)
    - [DBSC(E) use cases](#dbsce-use-cases)
      - [IDP is RP and Calls Public Local Key Helper](#idp-is-rp-and-calls-public-local-key-helper)
      - [IDP Calls Public Local Key Helper](#idp-calls-public-local-key-helper)
      - [IDP Calls Private Local Key Helper](#idp-calls-private-local-key-helper)
    - [Cleanup of the binding keys and their artifacts](#cleanup-of-the-binding-keys-and-their-artifacts)

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

While the original DBSC proposal enables browsers to bind session cookies to a device, it still remains vulnerable to "on device" malware. Such a malware, if present on the device, can inject its own binding keys when the DBSC session is established during any signin operaton. If a DBSC session is already established when the malware gains access to the system, the malware can force a new signin operation, and potentially hijack all subsequent sessions. Any upcoming sessions after this, even with DBSC, will not be reliable. Hence a temporary malware presence in the system can result in permanent session compromise in certain cases.

DBSC(E) aims to mitigate this risk by introducing the concept of once-in-a-lifetime protected [device registration](#device-registration) operation and binds all the future sessions to binding keys that can be cryptographically proven to be on the same device. DBSC(E) is able to provide this risk mitigation if the device registration is performed when there is no malware on the device (a state referred to as ["clean room"](#device-registration-client)) e.g. an organization registering a device before giving a device to an employee. Device registration is also expected to be a once-in-a-lifetime protected operation, hence the user will not be required to perform this operation again, reducing opportunities for malware to compromise a user session. 

Therefore, if a device registration is executed in a clean room and precedes any sign in sessions, DBSC(E) makes it impossible for a malware to bind session cookies to malicious binding keys during a sign in operation that implements DBSC(E). However, it is to be noted that, DBSC(E) doesn't protect a given session if the malware is present during the device registration or if the malware is persistent on the device and uses device-bound sessions to exfiltrate application data rather than the sessions.

## How does it integrate with DBSC?

DBSC(E) is not intended to be a separate proposal from DBSC, it is rather building on existing DBSC, and adds the binding specific details to the protocol. It is expected that the DBSC(E) proposal will be integrated into the DBSC proposal in the specification. In the high-level design, we have folded the DBSC proposal into the end to end flow. Please read the [DBSC proposal](https://githuub.com/wicg/dbsc) before you proceed.

Before we get into the specifics, we will introduce the terminology and design specifics for the key generation and validation below.

## Terminology

### Browser

In this document, "Browser" refers to the functionality in a web browser that is responsible for the DBSC protocol. This functionality will be implemented by Edge, Chrome (or their common engine), and other browsers that choose to implement DBSC/DBSC(E).

### Relying Party (RP)

A web application that uses DBSC(E) protocol for cookie binding. This is referred to as `server` in the original [DBSC design](https://githuub.com/wicg/dbsc).

### Identity Provider (IdP)

IdP is an authentication server that can be either external to the Relying Party or part of the Relying Party. Eg: Office.com authenticating with Microsoft Entra ID (external IDP) or google.com authenticating with google (no separate IDP). Note: The protocol doesn't change if the IDP is part of the Relying Party, except that some redirects between the IdP and the RP can be skipped or implemented by other means. In the original [DBSC design](https://githuub.com/wicg/dbsc), IDP and RP are the same entity, and referred to as `server`.

### Device Registration Client

This is a pre-requisite for DBSC(E) to work.

Device Registration is a process where the user or administrator registers the device with the IdP and is expected to be a once-in-a-lifetime protected operation.

The device registration establishes trust between the device and a service that maintains a directory of all devices. This document does not cover the protocol of device registration, but it assumes that during device registration, some asymmetric keys are shared between the client and the service, typically a device key, [attestation key](#attestation-key) and some other keys necessary for the secure device communication. A client software component that performs the device registration is called a _device registration client_. As mentioned above, the key assumption in DBSC(E) is that device registration happened in a clean room environment, and it is the responsibility of the device owner to ensure this. 

A clean room enviroment is a reliable, malware and exploit free state of a system. Examples can be: 
- New device from the factory connected to a secure network during registration. 
- Company issued devices configured by admin for the employees.
- A malware free device installing browser in a secure network.

One device registration client can manage multiple devices on the same physical device. There also can be multiple device registration clients on the same device. For example, a user can register a device to their organization as an employee and can simultaneously register the same device for a vendor organization. Both these registrations can either be managed by a single device registration client or through separate device registration clients depending on the implementation and separation between these organizations.

The device registration client can be owned and supported by:

- Operating system - the device gets registered when the OS is installed.
- Browser - the device gets registered when the browser is installed.
- Device management software (MDM provider) - the device gets registered when the MDM is enrolled.
- Third-party software vendor - the device gets registered according to the vendor rules.

DBSC(E) aims to support most of these scenarios. DBSC(E) does not define the device registration protocol, but is only concerned that the device registration is executed in a "clean room" and the management of the generated keys to prove device binding.

The device registration is expected to be a once-in-a-lifetime protected operation, and the user is expected to perform this operation with a clean room environment.

![DeviceRegistration](./DeviceRegistration.svg)

### Local Key Helper

DBSC(E) introduces the concept of `Local Key Helper`.

**Local Key Helper** is an integral part of the the **Device Registration Client**, a software interface responsible for the DBSC Key management. *Local key helper* can be public or private and is expected to be either shipped as a part of a given enterprise framework (with the IDP/OS) or can be installed by a provider in compliance with the protocol expanded below. DBSC(E) defines browser interaction with _Local key helpers_ for each platform. Those details are outlined in [KeyGeneration.md](./KeyGeneration.md).

From the deployment point of view there are two types of local key helpers: _private_ and _public_

- _Public local key helper_: Expected to have a well-documented API and can be used by any Identity Provider (IdP). Typically owned by a provider different from the IdP, communicates with the IdP as defined in DBSC(E) protocol. 
- _Private local key helper_ : Is specific to an IdP. Can be only used by a specific IDP that owns the implementation and will have a private protocol to communicate with the IdP. 

The Local Key Helper is responsible for:

- Generation of the [binding key](#binding-key) and producing [binding statements](#binding-statement) (see below).
- Producing signatures with the binding key.
- Cleanup of the [binding key](#binding-key) and its artifacts (when the user clears the browser session or the key is unused for a long time).

#### Platform Requirements

This section prescribes the browser discovery process of a given local key helper for a few well known platforms:

- [Windows](./LocalKeyHelper-Windows.md)
- [MacOS](./LocalKeyHelper-Mac.md)
- [Android](./LocalKeyHelper-Android.md)

Note: Above are platform specifications for Local Key Helpers that can be used for DBSC(E) key generation. Any vendor can ship their local key helper in compliance with the DBSC(E).

### Attestation Service

Attestation service is responsible for
- verifying that the binding key is issued by the expected device.
- providing the attestation of the key to the IdP. 

The attestation service can be owned by the IdP or a third party. DBSC(E) relies on the attestation service to validate the binding statement and ensure that the binding key and the device key belong to the same device. An attestation service can be part of the IdP, or a separate service. 

This document does not define the implementation details of the Attestation Service. It defines the artifacts which are generated during the [Device Registration](#device-registration-client) and are necessary for the Attestation Service to validate the binding statement.

#### Key Generation Specifics

There are three artifacts that are generated during/after the device registration process, which are used to prove the device binding. These are the [attestation key](#attestation-key), the [binding key](#binding-key), and the [binding statement](#binding-statement).

##### Attestation Key

An _attestation key_ is generated during the device registration process and has the following properties:

    1. It signs only the private/other keys that reside in the same secure enclave as the attestation key.
    2. It cannot sign any external payload, or if it signs, it cannot generate an output that can be interpreted as an attestation statement.

In addition, the _attestation key_ can be uploaded only once to the backend at the moment of device registration, in the clean room, and is seldom changed unless the device loses it (could be due to key rotation or similar operations).

The _attestation key_, hence, can be used to attest that the [binding key](#binding-key) belongs to the same device as the _attestation key_, by signing the public part of the _binding key_ (with the _attestation key_) and generating an _attestation statement_. Depending on the specific implementation, this _attestation statement_ itself can be a _binding statement_, or it can be sent to an [attestation service](#attestation-service) to produce the final _binding statement_. 

Note: _Attestation Key_ is also referred as _AIK_ in the document in some of the flow diagrams below.

##### Binding Key

A _binding key_ is an asymmetric key pair that is used to bind an auth cookie. It is identified by a _KeyId_ and it is the responsibility of the browser to remember _KeyId_ mapping to _Local Key Helper_ and _RP_ and to use it for DBSC signatures and key management. As there could be multiple `devices` on a single physical device and or many device registration clients on a single device (mentioned [above](#device-registration-client)), the key mapping is expected to be managed by the Browser and the Local Key Helper.

This _binding key_ for DBSC(E) is similar to the artifact defined in the DBSC proposal [here](https://github.com/WICG/dbsc?tab=readme-ov-file#maintaining-a-session). It is expected to be cryptographically attested by the [attestation key](#attestation-key) where the _attestation key_ is expected to be created in a secure enclave (TPM or Keyguard) on the same device. The original DBSC proposal(https://github/wicg/dbsc) does not guarantee the _binding key_ validation to be attack free, as it can be generated by a malware, [if the malware is present on the device](#why-dbsce). However, in DBSC(E), as long as the [device registration process is executed with a clean room environment](#device-registration), _binding key_ can be mapped to a specific device and the bound session is protected from any malware trying to infilterate it.

##### Binding Statement

Additonal to the _binding key_, the local key helper also generates a _binding statement_, a statement that asserts the binding key was generated on the same device as the device key. Details on how this statement is issued are out of scope for this document. However, the validation of the binding statement is a key building block of the DBSC(E) protocol.

The validation of the _binding statement_ authenticates the device by using device ID to find the corresponding _attestation key_. The validation component verifies the _attestation statement_ or the _binding statement_ (difference illustrated [in the section above](#attestation-key)), and it can understand that such a statement cannot be generated, unless the private key resides in the same secure enclave when signed by the _attestation key_. Hence, a valid _attestation statement_ or _binding statement_ means that both the _attestation key_ and the _binding key_ belong to the same device. The validation component can be part of the _attestation service_ for public local key helpers, or part of an IdP for private local key helpers.

This is not the only way to ensure that the _binding key_ and the _device key_ belong to the same device, and having the _attestation key_ and the _attestation service_ is not mandatory for producing a _binding statement_. That is why the protocol specifics of checking the binding are out of scope for this document. The focus of DBSC(E) is to only establish important properties of the _binding statement_.

Binding statements can be long-lived or short-lived. 
- If an IdP performs fresh device authentication outside of DBSC(E) integration at the time of _binding key_ validation, then the _binding statement_ can be long lived, as the IdP ensures the device identity through other mechanisms. 
- IdPs that do not perform proof of possession of the device, the ones that use public local key helpers, must use short-lived binding statements. Otherwise, the attacker will be able to bind the victim's cookies to malicious keys from a different machine. A short-lived binding statement must have an embedded nonce sent by the IdP to validate that it is a fresh binding statement, hence preventing the attack. 

## High-Level Design

DBSC(E), if enabled for a given enteprise, specifies the generation of the cryptographic artifacts (keys and binding info) before a sign in session is established. By enabling the browser to invoke specific APIs based on an enterprise policy, it allows enterprises to add to the existing key generation. It also allows them to place stricter restrictions on specific sessions, hence providing the flexibility to secure a session appropriately.

The high-level design is divided into two parts:

1. Key generation and validation before the session starts (DBSC(E) is focused on this part).
2. DBSC protocol applied with the generated keys (DBSC is focused on this part).

Since we want to integrate DBSC(E) with the original design and make it as widely applicable as possible for all enterprise users, we are adding high-level design for the most possible combinations in this document. The intent is to have a specification that can be implemented by any browser vendor, and can be used by any IdP, and any Local Key Helper. As we cover different use cases DBSC(E) can be applied for, we differentiate between private and public local key helpers, since there are implications to the protocol based on the type of local key helper. For example, we expect well establised IdPs like Microsoft, Github to ship their own private local key helpers. DBSC(E) protocol also provides multiple extension points that simplify and improve performance for specific scenarios. The [DBSC(E) use cases](#dbsce-use-cases) section expands on some of these scenarios.

DBSC(E) (in contrast with DBSC):

![DBSC(E) Highlevel Design](<./DBSC(E).svg>)

Highlights:

Note: All references to RP, IDP are equivalent to `server` in the original [DBSC design](https://github.com/wicg/dbsc).

1. **Pre-Session initiation with special headers (steps 1-2):** When a user starts a sign-in process, or initiates a session, the webpage initiating the session sends special headers `Sec-Session-GenerateKey` and `Sec-Session-HelperIdList` to the browser in response, to indicate that the session is expected to be DBSC(E) compliant.

   - The `Sec-Session-GenerateKey` header contains the URL of the server(RP), the URL of the IdP (authentication service in most cases - which is optional for consumer use cases), a _nonce_ and any extra parameters that the IdP wants to send to the Local Key Helper. _nonce_ is essential to prevent replay of cached binding key/binding statements from a different device (proof of posession) and to prevent the clock-skew between the IdP and the Local Key Helper. 
      - For all _public local key helpers_, e.g., Contoso's IDP calling Fabrikam's Local key helper, _nonce_  must be _shortlived_. If _binding statement_ is not shortlived, it is possible for the attacker to generate the _binding statement_ and the _binding key_ from a device controlled by the attacker and use them to bind the victim's cookies to the malicious _binding key_. The enforcement of a shortlived binding statement is achieved through _nonce_. 
      - The allowance for long lived _binding statement_ is possible with _private local key helpers_ where the IDP can use other means to establish fresh proof of posession of the device. This is covered in detail in [later sections](#idp-calls-private-local-key-helper).
      - _nonce_ also helps prevent the clock skew between servers where IDP and Attestation servers are from different vendors. Since the _nonce_  sent by the IDP is embedded in the _binding statement_, the IDP will be able to validate _nonce_ to ensure the _binding statement_ is issued recently.
      - _nonce_ is generated by the IdP/RP as a part of the request, is a random number that MUST be unique, MUST be time sensitive and MUST be verifiable by the issuer. 
   - The `Sec-Session-HelperIdList` header contains an ordered list of helper IDs that the browser can use to generate the key. The browser typically picks the first available and enabled `HelperId` from the list. The browser will then call the Local Key Helper with the `HelperId` to generate the key.

1. **Key and Binding Statement Generation (steps 3-7):** The Local Key Helper generates the key and the binding statement. AIK refers to the `Attestation Key` described [above](#attestation-key). The binding statement is expected to contain the _nonce_ sent by the IdP, the thumbprint of the public key, and any extra claims that the IdP wants to send.

   - Format of the Binding Statement: We expect the _binding statement_ will be a `string`, as we want to keep the format open to allow for platform-specific optimizations. However, the validation of the _binding statement_ is prescribed to include _nonce_ and the thumbprint of the public key. The _binding statement_ is expected to be shortlived to prevent forgery of the binding statement from a different device. More details on _binding statement_ can be found [here](#binding-statement).
   - The `extra claims` is a provision added for specific IdPs or Local Key Helper vendors to add any additional information to the _binding statement_. It is intentionally left undefined, and can be customized.

1. **Sign In/Session Initiation (steps 8-9):** The _binding statement_, with the `KeyId` is expected to be returned to the IdP with a new header, `Sec-Session-Keys`. The IdP will [validate](#binding-statement) the signature on the _binding statement_, _nonce_ and stores the thumbprint of the public key. Once the validation succeeds, the IdP will proceed with the sign-in ceremony (optionally generate auth tokens if the RP and IdP are separate, illustrated [below](#idp-is-rp-and-calls-public-local-key-helper), that embed the thumbprint of the public key). The `KeyId` is expected to be returned to the RP/IdP, and the IdP will use the `KeyId` to identify the key to be used for the session.

1. **SignIn Succeeds with binding (steps 10-14)**: At this point, all DBSC(E) specific steps are completed. The server returns signed in content with a special header response to the browser: `Sec-Session-Registration` indicated the session is expected to be DBSC compliant. All steps further are as per the original DBSC proposal with two caveats: The _local key helper_ is called for JWT generation and the additional params introduced for DBSC(E) customization can be optionally added.

### DBSC(E) use cases

This section expands on the [generic design](#high-level-design) to address different enterprise use cases:

#### IDP is RP and Calls Public Local Key Helper

This is the same use case elaborated in the high level design [above](#high-level-design). However, we have separated the IdP/RP components of the server that applies to certain enterprise and consumer use cases in the below diagram.

![IDPSameAsRP-CallsPublicLocalKeyHelper](./IDPSameAsRP-CallsPublicLocalKeyHelper.svg)

Highlights:

- Auth tokens are not mentioned in the DBSC(E) high level design to simplify the over all scenario. However, most signin/access operations will often make use of auth tokens and issue cookies based on those tokens. In such case, the IdP can generate the auth tokens and embed the thumbprint of the public key in the token.
- The tokens can be delivered to the RP directly through an API instead of a header based response.
- The tokens also contain the thumbbrint of the public key, and the RP must validate the thumbprint from the token against the thumbprint from the JWT. The RP will also embed the thumbprint in the cookie for subsequent session validation.

#### IDP Calls Public Local Key Helper

In many use cases, it is also a valid to have separate servers as [RP](#relying-party-rp) and [IdP](#identity-provider-idp). We address the use case where these are separate entities and probably from different vendors below.

For easy mapping with the existing DBSC proposal, please note:

- Steps 1-16 specify the key generation process for a public local key helper.
- Steps 17-29 are [DBSC](https://github.com/wicg/dbsc), added for completeness.

![IDPCallsPublicLocalKeyHelper](./IDPCallsPublicLocalKeyHelper.svg)

### Cleanup of the binding keys and their artifacts

For the health of the operating system and user privacy, it is important to clean up binding keys and their artifacts. If the number of keys grows too high, the performance of the OS can degrade.

The cleanup can occur:

- When cookies are expired.
- On demand, when the user decides to clear browser cookies.
- Automatically, when a key hasn't been used for a long time (N days) to keep the OS healthy.
