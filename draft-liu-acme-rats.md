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
 -
    fullname: Michael Richardson
    organization: Sandelman Software Works Inc
    email: mcr+ietf@sandelman.ca

normative:
  RFC8555:

informative:
  I-D.ietf-lamps-csr-attestation: CSRATT
  I-D.draft-moriarty-rats-posture-assessment: RATSPA

--- abstract

This document describes an approach where ACME Client can prove possession of a Remote Attestation Result to an ACME Server.

--- middle

# Introduction

ACME {{RFC8555}} is a standard protocol for issuing and renewing certificates automatically.
ACME is been a key component in making certificate issuance easily automated, and that has permitted widespread adoption of HTTPS.  The ACME protocol includes mechanisms by which a client may prove it is authorized for particular identities to be included in the certificate. These identifers are typically DNS names, but not exclusively.  ACME includes an authorization system consisting of different challenges that prove ownership to identifiers like domain name or email address.

However, ACME does not put any restrictions on the security posture of the device requesting a certificate, neither at initial issuance time, nor on an ongoing basis.

In this document, it is proposed to have ACME Servers evaluate remote attestation results as part of the certificate issurance and renewal process.
The posession of a valid, fresh, remote attestation result by the ACME client, for instance an EAT (entity attestation token) constitutes proof that the ACME client has had a recent security posture evaluation.

Possession of an attestion result presupposes that the client has communicated with a verifier, and further, that the attestation result has been produced by a verifier that produces attestation results that the ACME server has been configured to trust.

An example use case includes enterprise scenarios where Network Operations Center (NOC) issue certificates to devices that are freshly appraised by the Security Operations Center (SOC), in order to help them work together.

# Terminology

{::boilerplate bcp14-tagged}

The terms Attester, Verifier, Relying Party, Evidence and Attestation Results are from {{RFC9334}}.

For ease of denotion, we omit the "ACME" adjective from now on, where Server means ACME Server and Client means ACME Client.

# ACME processes

As explained in {{RFC8555, Section 4}}, ACME consists of the following steps:

1. Account Creation (newAccount, newNonce)
2. Order creation (newOrder)
3. Validation (newAuthz)
4. Certificate Issuance (newOrder)

While it might appear that Remote Attestation should occur at the authorization step, the ACME authorization process is about proving ownership of identities that can go into the resulting certificates.
This includes dns-01, and http-01 challenges for DNS FQDN, but also email address {{?RFC8823}}, and also DTN IDs.
In each case, the client picks a single challenge to respond to, and the result of the challenge provides authorization for the client to speak for that particular identifier.

Remote Attestation occurs at an entirely different stage of the protocol, and provides a completely different piece of information to the ACME server.
Remote Attestation occurs during the Account Creation process as part of the authentication protocol: {{RFC8555, Section 7.3.4}}.

## Passport Model for ACME

In the passport model, the (ACME) Client has to reach out to a verifier and provide Evidence to the Verifier.
The Verifier responds with Attestation Results which is then included in the newAccount request using a new header attribute: "ar".

It is not necessary for standardization of this trustworthy extension to standardize the specifics of how the Attester/Verifier communication is done.


## Background-Check model for ACME

RATS-02 Challenge works with the Background Check Model of RATS.

TODO: After the Client gets the "token", it include "token" as part of its RATS Evidence, appraise again. The new attestationResult now has a "token" claim. The retrival process is same.

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
