---
title: Automated Certificate Management Environment (ACME) rats Identifier and Challenge Type
abbrev: acme-rats
category: std

docname: draft-liu-acme-rats-00
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
  I-D.ietf-lamps-csr-attestation: CSRATT
  I-D.draft-moriarty-rats-posture-assessment: RATSPA
  I-D.draft-moriarty-rats-posture-assessment: DEVATT

--- abstract

This document provides a generic approach for an Automated Certificate Management Environment (ACME) Server to establish confidence over an ACME Client before issuing ACME certificates to it. 

--- middle

# Introduction

ACME {{RFC8555}} is a standard protocol for issuing and renewing certificates automatically, widely used in the Internet scenario, help an ACME Client prove its ownership to an identifier like domain name or email address.

In order to prevent issuing certificates to malicious devices, a few works are ongoing in the LAMPS and RATS WG.

- {{?I-D.ietf-lamps-csr-attestation}} define trustworthy claims about device's platform generating the certification signing requests (CSR) and the private key resides on this platform.
- {{?I-D.draft-moriarty-rats-posture-assessment}} define a summary of a local assessment of posture for managed systems and across various layers.

In this document, we propose an approach using which an ACME Server validates generic properties of an ACME Client, not limited to the identity of a device as defined in {{?I-D.draft-acme-device-attest}}. This draft should be considered as a generalization of {{?I-D.draft-acme-device-attest}}.

For ease of denotion, we omit the "ACME" adjective from now on, where Server means ACME Server and Client means ACME Client.

# rats identifier

TODO: is rats-identifier still necessary?

type (required, string):
: The string "rats".

value (required, string):
: The identifier itself.

# rats challenge type: challenge object and response

Challenge Object:

type (required, string):
: The string "rats".

url (required, string):
: The URL that the Client post its response to.

token (required, string):
: Same as Section 8.3 of RFC8555. A background check evidence MUST include this in the evidence.

attestType (required, string):
: Values from the registry defined by Section 6.2 of {{?I-D.ietf-lamps-csr-attestation}}

attestClaimsHint (required, string):
: attest claims array requested in this challenge object.

An example object:

~~~~~~~~~~
{
  "challenges": [
    {
      "type": "rats",
      "url": "https://example.com/acme/chall/prV_B7yEyA4",
      "status": "pending",
      "token": "DGyRejmCefe7v4NfDGDKfA",
      "attestType": ["tcg-dice-manifest-evidence", "tcg-dice-UCCS-evidence],
      "attestClaimsHint": ["OS patch level", "FIPSmode"]
    },
  ],
}
~~~~~~~~~~

An example response:

~~~~~~~~~~
  {
    "protected": base64url({
      "alg": "ES256",
      "kid": "https://example.com/acme/acct/evOfKhNU60wg",
      "nonce": "SS2sSl1PtspvFZ08kNtzKd",
      "url": "https://example.com/acme/chall/Rg5dV14Gh1Q"
    }),
    "payload": base64url({<CMW>}),
    "signature": "Q1bURgJoEslbD1c5...3pYdSMLio57mQNN4"
  }
~~~~~~~~~~

# Example use case -- enterprise access

ACME-rats extension can be used in an enterprise network scenario. An Endpoint Detection and Response (EDR) software and Security Operations Center (SOC) is responsible for checking the security status of an accessing end device. If the device passed the most recent security checks, EDR/SOC should issue fresh attestation results (in which case the CMW in the challenge-response is an Attestation Result). A Network Operations Center (NOC) could use ACME-rats to issue short-lived certificates to only devices with fresh attestation results. Otherwise, EDR/SOC should refuse to issue (new) attestation results thus access privileges. In this way, the NOC can issue network access only to devices that are continuously monitored and have no known security risks up-to-date. SOC can also have flexible security policies and decide what to check.

# Security Considerations

TODO Security

# IANA Considerations

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
