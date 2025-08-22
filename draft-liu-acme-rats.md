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
  RFC9334:
  CMW: I-D.draft-ietf-rats-msg-wrap
informative:
  CSRATT: I-D.ietf-lamps-csr-attestation
  RATSPA: I-D.draft-moriarty-rats-posture-assessment
  I-D.draft-bweeks-acme-device-attest-01: device-attest-01
  letencrypt:
    target:    https://www.eff.org/deeplinks/2023/08/celebrating-ten-years-encrypting-web-lets-encrypt
    title: "Celebrating Ten Years of Encrypting the Web with Let’s Encrypt"
    author:
      org:
      - "Electronic Frontier Foundation"
    date: 2025-08-20

--- abstract

This document describes an approach where an ACME Server can challenge an ACME Client to provide a Remote Attestation Evidence or Remote Attestation Result in any format supported by the Conceptual Message Wrapper.

The ACME Server can optionally challenge the Client for specific claims that it wishes attestation for.

--- middle

# Introduction

ACME {{RFC8555}} is a standard protocol for issuing and renewing certificates automatically.
ACME clients needing a certificate from a certification authority connect to the ACME server, provide a proof of control of a desired identity, and then receive a certificate with that identity in it.

These identities become part of the certificate, usually a Fully Qualified Domain Name (FQDN) that goes into the Subject Alt Name (SAN) for a certificate.
Prior to ACME, the authorization process of obtaining a certificate from an operator of a (public) certification authority was non-standard and ad-hoc.
It ranged from sending faxes on company letterhead to answering an email sent to a well-known email address like hostmaster@example.com, evolving into a process where some randomized nonce could be placed in a particular place on the target web server.
The point of this process is to prove that the given DNS FQDN was controlled by the client system.

ACME standardized the process, allowing for automation for certificate issuance.
It has been a massive success: increasing HTTPS usage from 27% in 2013 to over 80% in 2019 {{letsencrypt}}.

While the process supports many kinds of identifiers: email addresses, DTN node IDs, and can create certificates for client use.
However, these combinations have not yet become as popular, in part because these types of certificates are usually located on much more general purpose systems such as laptops and computers used by people.

The concern that Enterprises have with the use of client side certificates has been the trustworthiness of the client system itself.
Such systems have many more configurations, and are often considered harder to secure as a result.
While well managed mutual TLS (client and server authentication via PKIX certificate) has significant advantages over the more common login via username/passowrd,  if the private key associated with a client certificates is disclosed or lost, then the impact can be more significant.

The use case envisioned here is that of an enterprise.  A Network Operations Center (NOC)
(which may be internal or external) will issue (client) certificates to devices that can prove via remote attestation that they are running an up-to-date operating system as well as the enterprise-required endpoint security software.

This is a place where Remote Attestation can offer additional assurance {{!RFC9334}}.
If the software on the client is properly designed, and is up to date, then it is easier to assure that the private key will be safe.

This can be extended to Bring Your Own Device (BYOD) by having those devices provide an equivalent Attestation Result.

This document defines an extension to ACME that allows an ACME server to received the signed (and fresh) Attestation Result.

ACME can presently offer certificates with multiple identities.
Typically, in a server certificate situation, each identity represents a unique FQDN that would be placed into the certificate as distinct Subject Alt Names (SAN).
For instance each of the names: example.com, www.example.com, www.example.net and marketing.example.com might be placed in a single certificate for a server that provides web content under those four names.

This document defines a new identity type, `trustworthy` that the ACME client can ask for.
The new `attestation-result-01` challenge is defined as a new method that can be used to authorize this identity.

For ease of denotion, we omit the "ACME" adjective from now on, where Server means ACME Server and Client means ACME Client.

## Attestation Results only

This document currently only defines the a mechanism to carry Attestation Results from the ACME client to the ACME server.
It limits itself to the Passport model defined in {{RFC9334}}.

This is done to simplify the combinations, but also because it is likely that the Evidence required to make a reasonable assessment includes potentially privacy violating claims.
This is particularly true when a device is a personal (BYOD) device; in that case the Verifier might not even be owned by the Enterprise,  but rather the device manufacturer.

In order to make use of the background check that Evidence would need to be encrypted from the Attesting Environment to the Verifier, via the ACME Server -- the Relying Party.
Secondly, in order for the ACME Server to be able to securely communicate with an Enterprise located Verifier with that Evidence, then more complex trust relationships would need to be established.
Thirdly, the Enterprise Verifier system would then have to expose itself to the ACME Server, which may be located outside of the Enterprise.
The ACME Server, for resiliency and loading reasons may be a numerous and dynamic cluster of systems, and providing appropriate network access controls to enable this communication may be difficult.

Instead, the use of the Passport model allows all Evidence to remain within an Enterprise,
and for Verifier operations to be more self-contained.

## Related work

{{CSRATT}} define trustworthy claims about the physical storage location of a key pair's private key.
This mechanism only relates to how the private key is kept.
It does not provide any claim about the rest of the mechanisms around access to the key.
A key could well be stored in the most secure way imaginable, but in order to use the key some command mechanism must exist to invoke it.

The mechanism created in this document allows certification authority to access the trustworthiness of the entire system.
That accessment goes well beyond how and where the private key is stored.
ACME uses Certificate Signing Requests, so there is no reason that {{CSRATT}} could not be combined with the mechanism described in this document.

{{RATSPA}} defines a summary of a local assessment of posture for managed systems and across various  layers.
The claims and mechanisms defined in {{RATSPA}} are a good basis for the assessment that will need to be done in order to satisfy the trustworthiness challenge detailed in this document.

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
: If the Server requires attestation of specific claims or properties in order to issue the requested certificate profile, then it MAY list one or more types of claims from the newly-defined ACME Attest Claims Hints registry defined in {{claimshints}}.

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
