---
stand_alone: true
category: std
submissionType: IETF
ipr: trust200902
lang: en
title: Automated Certificate Management Environment (ACME) rats Identifier and Challenge Type
abbrev: acme-rats
docname: draft-liu-acme-rats-latest
area: "Security"
workgroup: "Automated Certificate Management Environment"
kw:
 - ACME
 - RATS
 - Zero Trust
venue:
  group: "Automated Certificate Management Environment"
  type: "Working Group"
  mail: "acme@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/acme/"
  github: "liuchunchi/draft-liu-acme-rats"
  latest: "https://liuchunchi.github.io/draft-liu-acme-rats/draft-liu-acme-rats.html"
author:
 -
    fullname: Chunchi Peter Liu
    organization: Huawei
    email: liuchunchi@huawei.com
 -
    fullname: Mike Ounsworth
    organization: Entrust Limited
    email: mike.ounsworth@entrust.com
 -
    fullname: Michael Richardson
    organization: Sandelman Software Works Inc
    email: mcr+ietf@sandelman.ca
normative:
  RFC8555:
  I-D.draft-ietf-rats-msg-wrap: CMW
informative:
  I-D.ietf-lamps-csr-attestation: CSRATT
  I-D.draft-moriarty-rats-posture-assessment: RATSPA
  I-D.draft-bweeks-acme-device-attest-01: device-attest-01

--- abstract

This document describes an approach where an ACME Server can challenge an ACME Client to provide a Remote Attestation Evidence or Remote Attestation Result in any format supported by the Conceptual Message Wrapper. The ACME Server can optionally challenge the Client for specific claims that it wishes attestation for.

--- middle

# Introduction

ACME {{RFC8555}} is a standard protocol for issuing and renewing certificates automatically, widely used in the Internet scenario, help an ACME Client prove its ownership to an identifier like domain name or email address.

In order to prevent issuing certificates to malicious devices, a few works are ongoing in the LAMPS and RATS WG.

- {{?I-D.ietf-lamps-csr-attestation}} define trustworthy claims about device's platform generating the certification signing requests (CSR) and the private key resides on this platform.
- {{?I-D.draft-moriarty-rats-posture-assessment}} define a summary of a local assessment of posture for managed systems and across various layers.

This document builds on {{I-D.draft-bweeks-acme-device-attest-01}} which provides a mechanism for WebAuthn attestations over ACME. This document is broader in scope to support a broad range of attestation formats.

In this document, we propose an approach where ACME Server MAY challenge the ACME Client to produce an Attestation Evidence or Attestation Result in any format that is supported by the RATS Conceptual Message Wrapper [CMW]. The ACME Server then checks if the ACME Clients presented a valid remote attestation evidence or remote attestation result, for instance, an EAT (entity attestation token). The ACME Server MAY perform any necessary checks against the provided remote attestation, as required by by the requested certificate profile; this conforms to the RATS concept of an Appraisal Policy.

This document defines a new ACME "rats" identifier and the challenge types "device-attest-02" and "device-attest-03" which are respectively use to challenge for a RATS background check and passport model type attestation. In this way, the CA / RA issues certificates only to devices that can provide an appropriate attestation result, indicating such device has passed the required security checks. By repeating this process and issuing only short-lived certificates to qualified devices, we also fulfill the continuous monitoring/validation requirement of Zero-Trust principle.

Some example use cases include an enterprise scenario where Network Operations Center (NOC) issue certificates to devices that can prove via remote attestation that they are running an up-to-date operating system as well as the enterprise-required endpoint security software. Another example is issuing S/MIME certificates to BYOD devices only if they can prove via attestation that they are registered to a corporate MDM and the user they are registered to matches the user for which a certificate has been requested.

For ease of denotion, we omit the "ACME" adjective from now on, where Server means ACME Server and Client means ACME Client.


# Extensions -- rats identifier

An rats identifier type represents a unique identifier to an attestation result. It extends a "rats" identifier type and a string value.

type (required, string):
: The string "rats".

value (required, string):
: The identifier itself.

The following steps are the ones that will be affected:

1\. newOrder Request Object - identifiers: During the certificate order creation step, the Client sends a /newOrder JWS request (Section 7.4 of {{RFC8555}}) whose payload contains an array of identifiers. The Client adds an rats identifier to the array.

An example extended newOrder JWS request:

~~~~~~~~~~
  {
    "protected": base64url({
      "alg": "ES256",
    }),
    "payload": base64url({
      "identifiers": [
        { "type": "rats", "value": "0123456789abcdef" },
      ],
    }),
    "signature": "H6ZXtGjTZyUnPeKn...wEA4TklBdh3e454g"
  }
~~~~~~~~~~

2\. Order Object - identifiers: After a newOrder request is sent to the Server, the Account Object creates an Order Object (Section 7.1.3 of {{RFC8555}}) with "rats" identifiers and values from Step 1.

An example extended Order Object:

~~~~~~~~~~
  {
    "status": "pending",

    "identifiers": [
      { "type": "rats", "value": "0123456789abcdef" },
    ],

    "authorizations": [
      "https://example.com/acme/authz/PAniVnsZcis",
    ],

    "finalize": "https://example.com/acme/order/T..fgo/finalize",
  }
~~~~~~~~~~

3\. Authorization Object - identifier: The Server creates an Authorization Object that has rats identifier (Section 7.1.4 of {{RFC8555}})

4\. Challenge Object - identifier: The Server creates a Challenge Object that has rats challenge type.

An example extended Authorization Object (that contains a Challenge Object):

~~~~~~~~~~
{
  "status": "pending",

  "identifier": {
    "type": "rats",
    "value": "0123456789abcdef"
  },

  "challenges": [
    {
      "type": "rats",
      "url": "https://example.com/acme/chall/prV_B7yEyA4",
      "status": "pending",
      "token": "DGyRejmCefe7v4NfDGDKfA",
    },
    {
      "type": "http-01",
      "url": "https://example.com/acme/chall/prV_B7yEyA4",
      "status": "pending",
      "token": "DGyRejmCefe7v4NfDGDKfA",
    }
  ],
}
~~~~~~~~~~


# Extensions -- rats challenge types

A rats challenge type help the Client prove ownership to its attestation result identifier. This section describes the challenge/response extensions and procedures to use them.

## device-attest-02 Challenge {#rats01}

device-attest-02 Challenge simply works with Passport Model of RATS. The corresponding Challenge Object is:

type (required, string):
: The string "device-attest-02".

url (required, string):
: The URL that the Client post its response to.

token (required, string):
: Same as Section 8.3 of RFC8555.

nonce (optional, string):
: If attestation freshness is required, then the Server MAY present a nonce which then MUST be echoed in the provided attestation. In some situations, the nonce will come from a separate RATS Verifier, and therefore needs to be a distinct value from the ACME token.

attestClaimsHint (optional, list of string)
: If the Server requires attestation of specific claims or properties in order to issue the requested certificate profile, then it MAY list one or more types of claims from the newly-defined ACME Attest Claims Hints registry defined in {{sec-claimshints}}.

The response sent to the url is:

~~~~~~~~~~
keyAuthorization = token || '.' || cmw
~~~~~~~~~~

where `cmw` MAY be either a CMW in JWT format, or a Base64 CMW in CWT format as per {{CMW}}.

## device-attest-03 Challenge {#rats02}

device-attest-03 Challenge works the same way as device-attest-02, but expects the Client to return RATS evidence in accordance with the Background Check Model of RATS.

# ACME Attest Claims Hint Registry {#claimshints}

In order to facilitate the Server requesting attestation of specific types claims or properties, we define a new registry of ACME Claims Hints. In order to preserve flexibility, the Claim Hints are intended to be generic in nature, allowing for the client to reply with any type of attestation evidence or attestation result that contains the requested information. As such, these values are not intended to map one-to-one with any specific remote attestation evidence or attestation result format, but instead they are to serve as a hint to the ACME Client about what type of attestation it needs to collect from the device. Ultimately, the CA's certificate policies will be the authority on what evidence or attestation results it will accept.

See {{iana-claimshints}} for the initial contents of this new registry.

# Example use case

## Enterprise WiFi Access

In enterprise access cases, security administrators wish to check the security status of an accessing end device before it connects to the internal network. Endpoint Detection and Response (EDR) softwares can check the security/trustworthiness statuses of the device and produce an Attestation Result (AR) if the check passes. ACME-RATS procedures can then be used to redeem a certificate using the AR.

With that being said, a more specific use case is as follows: an enterprise employee visits multiple campuses, and connects to each one's WiFi. For example, an inspector visits many (tens of) power substations a day, connects to the local WiFi, download log data, proceed to the next and repeat the process.

Current access solution include: 1. The inspector remembers the password for each WiFi, and conduct the 802.1X EAP password-based (PAP/CHAP/MS-CHAPv2) authentication. or 2. an enterprise MDM receives the passwords and usernames over application layer connection from the MDM server, and enter them on user's behalf. While Solution 1 obviously suffer from management burdens induced by massive number of password pairs, and password rotation requirements, the drawback of Solution 2 is more obsecure, which include:

a. Bring Your Own Device (BYOD) situation and MDM is not available.
b. Password could risk leakage due to APP compromise, or during Internet transmission. Anyone with leaked password can access, without binding of trusted/usual devices.
c. The RADIUS Client/Access Point/Switch is not aware of the identity of the accessing device, therefore cannot enforce more fine-grained access policies.

An ideal user story is: 
1. When the inspector is at base (or whenever the Remote Attestation-based check is available), he get his device inspected and redeem a certificate using ACME-RATS.
2. When at substation, the inspector authenticate to the WiFi using EAP-TLS, where all the substations have the company root CA installed.
2*. Alternatively, the Step 2 can use EAP-repeater mode, where the RADIUS Client redirects the request back to the RADIUS Server for more advanced checks.

# Security Considerations

TODO Security

# IANA Considerations

IANA is requested to open a new registry, XXXXXXXX

Type: designated expert

The registry has the following columns:

- Claim Hint: the string value to be placed within an ACME device-attest-02 or device-attest-03 challenge.
- Descryption: a descryption of the general type of attestation evidence or attestation result that the client is expected to produce.

The initial registry contents is shown in the table below.

| Claim Hint       | Description                  |
|------------------|------------------------------|
| FIPS_mode        | Attestation that the device is currently booted in FIPS mode.                           |
| OS_patch_level   | Attestation to the version or patch level of the device's operating system.             |
| sw_manifest      | A manifest list of all software currently running on the device.                        |

## ACME Attest Claims Hint Registry {#iana-claimshints}

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
