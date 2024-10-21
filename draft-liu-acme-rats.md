---
title: Automated Certificate Management Environment (ACME) rats Identifier and Challenge Type
abbrev: acme-rats
category: 

docname: draft-liu-acme-rats-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Security"
workgroup: "Automated Certificate Management Environment"
keyword:
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

normative:
  RFC8555:

informative:
  I-D.ietf-lamps-csr-attestation-10: CSRATT
  I-D.draft-moriarty-rats-posture-assessment: RATSPA

--- abstract

This document describes an approach where ACME Client can prove possession of a Remote Attestation Result to an ACME Server.

--- middle

# Introduction

ACME {{RFC8555}} is a standard protocol for issuing and renewing certificates automatically, widely used in the Internet scenario, help an ACME Client prove its ownership to an identifier like domain name or email address.

In order to prevent issuing certificates to malicious devices, a few works are ongoing in the LAMPS and RATS WG.
- {{?I-D.ietf-lamps-csr-attestation-10}} define trustworthy claims about device's platform generating the certification signing requests (CSR) and the private key resides on this platform.
- {{?I-D.draft-moriarty-rats-posture-assessment}} define a summary of a local assessment of posture for managed systems and across various layers.

In this document, we propose an approach where ACME Server checks if the ACME Clients possess a valid remote attestation result, for instance, EAT (entity attestation token). We define a new ACME "rats" identifier and "rats" challenge type for ACME Clients to prove their possession of EAT. In this way, we (as network administators) issue certificates only to devices that have a fresh attestation result, indicating such device has passed the most up-to-date security checks. By repeating this process and issue only short-lived certificates to qualified devices, we also fulfill the continuous monitoring/validation requirement of Zero-Trust principle.

The example use case include an enterprise scenario where Network Operations Center (NOC) issue certificates to devices that are freshly appraised by the Security Operations Center (SOC), in order to help them work together.

For ease of denotion, we omit the "ACME" adjective from now on, where Server means ACME Server and Client means ACME Client.


# Extensions -- rats identifier

An rats identifier type represents a unique identifier to an attestation result. It extends a "rats" identifier type and a string value.

type (required, string):
: The string "rats".

value (required, string):
: The identifier itself.

> REMOVE BEFORE SUBMISSION: Exact format of value? URI? hash of EAT? ueid/eat_nonce?

The following steps are the ones that will be affected:

1. newOrder Request Object - identifiers: During the certificate order creation step, the Client sends a /newOrder JWS request (Section 7.4 of {{RFC8555}}) whose payload contains an array of identifiers. The Client adds an rats identifier to the array.

An example extended newOrder JWS request:

```
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
```

2. Order Object - identifiers: After a newOrder request is sent to the Server, the Account Object creates an Order Object (Section 7.1.3 of {{RFC8555}}) with "rats" identifiers and values from Step 1.

An example extended Order Object:
```
{
  "status": "pending",

  "identifiers": [
    { "type": "rats", "value": "0123456789abcdef" },
  ],

  "authorizations": [
    "https://example.com/acme/authz/PAniVnsZcis",
  ],

  "finalize": "https://example.com/acme/order/TOlocE8rfgo/finalize",
}
```

> REMOVE BEFORE SUBMISSION: other ways to complete authorization step (7.1.3 of {{RFC8555}}): a. Pre-authorization (authz) b. External (RATS) account binding. These can allow background-check model of RATS. In mode a, the Server is a Verifier and the Client is the Attester. The authz process contains RATS process. In mode b. I havent considered that :P

3. Authorization Object - identifier: The Server creates an Authorization Object that has rats identifier (Section 7.1.4 of {{RFC8555}})
4. Challenge Object - identifier: The Server creates a Challenge Object that has rats challenge type.

An example extended Authorization Object (that contains a Challenge Object):
```
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
```


# Extensions -- rats challenge type

A rats challenge type help the Client prove ownership to its attestation result identifier. This section describes the challenge/response extensions and procedures to use them.

## RATS-01 Challenge {#rats01}

RATS-01 Challenge simply works with Passport Model of RATS. The corresponding Challenge Object is:

type (required, string):
: The string "rats-01".

url (required, string):
: The URL that the Client post its response to.

token (required, string):
: Same as Section 8.3 of RFC8555.

The response sent to the url is:

~~~~~~~~~~
keyAuthorization = token || '.' || base64url(attestationResult)
~~~~~~~~~~

where the attestationResult is the entire EAT (in JWT format). The ACME Server verifies the attestationResult. If pass, set Order Object and Authorization Object's "status" Object to "valid", otherwise "invalid".

> REMOVE BEFORE SUBMISSION: Actually, this usage might be tricky. 1. the original keyAuthorization string is token || '.' || base64url(Thumbprint(accountKey)). I replaced Thumbprint(accountKey) with attestationResult, no hash, which lacks a proof of possession to the accountKey, and attestationResult is plaintext. Should I construct a proper JSON/JWS Object, where the payload contains the attestationResult? 2. In the original RFC8555, this response string is not posted directly to the "url". Instead it post this response to its own server path and send only an empty {} to the url, notifying the Server it is ready to fetch. The server then does a canonical verification. (8.3 of {{RFC8555}})


## RATS-02 Challenge {#rats02}

RATS-02 Challenge works with the Background Check Model of RATS.

TODO: After the Client gets the "token", it include "token" as part of its RATS Evidence, appraise again. The new attestationResult now has a "token" claim. The retrival process is same.

> REMOVE BEFORE SUBMISSION: There might be different ways to prove ownership to rats identifiers. It really depends on what exactly does this "rats" identifier represent-- an attestation result (hash or EAT, eat_nonce, ...), or a RATS attester identity (ueid, ...).  As for -00, it only represents an attestation result.

# Reusing HTTP challenge type {#http}

We can also reuse http-01 challenge type in Section 8.3 of {{RFC8555}}. This changes steps used in {#rats01}.

1. The Client creates the keyAuthorization string defined in {#rats01}.
2. The Client provisions the keyAuthorization string in the resource path defined in the original http-01 challenge -- "/.well-known/acme-challenge/", followed by the "token".
3. The Client sends an empty object ({}) to the url, notifying the Server it is ready to fetch.
4. The Server fetches the keyAuthorization string from the resource path. Verifies the "token", retrive the attestationResult.


# Example use case -- enterprise access

In an enterprise network scenario, it is hard to coordinate Security Operations Center (SOC) and Network Operations Center (NOC) to work together due to various of reasons:

1. Integration/compatibility difficulty: Integrating SOC and NOC requires plenty of customized, case-by-case developing work. Especially considering differrnt system vendors, system versions, different data models and formats due to different client needs... Let alone possible updates.
2. Conflict of duties: NOC people do not want SOC people to interfere with their daily work, and so do SOC people. Also, NOC people may have limited security knowledge, and SOC people vice versa. Where to draw the line and what is the best tool to help them collaborate is a question.

This work proposes a way to help SOC and NOC work together, with separated duties (to avoid conflict) and ease of working together (proper abstraction).

An Endpoint Detection and Response (EDR) software and Security Operations Center (SOC) is responsible for checking the security status of an accessing end device. If the device passed latest security checks, EDR/SOC should issue fresh attestation results (consider as a security clearance). Otherwise, EDR/SOC should refuse to issue (new) attestation results. A Network Operations Center (NOC) could use ACME to issue short-lived certificates to only devices with fresh attestation results. In this way, the NOC can follow a Zero-Trust philosophy and issue network access to only devices that are continuously monitored and have no known security risks up-to-date. SOC can also have flexible security policies and decide what to check.

# Security Considerations

TODO Security

# IANA Considerations

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
