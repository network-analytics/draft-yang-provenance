---
title: "Applying COSE Signatures for YANG Data Provenance"
abbrev: "yang-data-provenance"
category: info

docname: draft-lopez-opsawg-yang-provenance-latest
submissiontype: IETF  # also: "independent", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Operations and Management"
workgroup: "Operations and Management Area Working Group"
keyword:
 - provenance
 - signature
 - COSE
 - metadata
venue:
  group: "Operations and Management Area Working Group"
  type: "Working Group"
  mail: "opsawg@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/opsawg/"
  github: "dr2lopez/yang-provenance"
  latest: "https://dr2lopez.github.io/yang-provenance/draft-lopez-opsawg-yang-provenance.html"
author:
 - name: Diego Lopez
   organization: Telefonica
   email: "diego.r.lopez@telefonica.com"
 - name: Antonio Pastor
   organization: Telefonica
   email: "antonio.pastorperales@telefonica.com"
 - initials: A.
   surname: Huang Feng
   fullname: Alex Huang Feng
   organization: INSA-Lyon
   email: "alex.huang-feng@insa-lyon.fr"

normative:
 RFC3688:
 RFC3986:
 RFC5277:
 RFC5322:
 RFC6020:
 RFC7950:
 RFC7951:
 RFC8340:
 RFC8641:
 RFC8785:
 RFC8949:
 RFC9052:
 RFC9254:
 I-D.ahuang-netconf-notif-yang: I-D.ahuang-netconf-notif-yang
 XMLSig:
  title: XML Signature Syntax and Processing Version 2.0
  target: https://www.w3.org/TR/xmldsig-core2/

informative:
 YANGmanifest: I-D.ietf-opsawg-collected-data-manifest

--- abstract

This document defines a mechanism based on COSE signatures to provide and verify the provenance of YANG data, so it is possible to verify the origin and integrity of a dataset, even when those data are going to be processed and/or applied in workflows where a crypto-enabled data transport directly from the original data stream is not available. As the application of evidence-based OAM automation and the use of tools such as AI/ML grow, provenance validation becomes more relevant in all scenarios. The use of compact signatures facilitates the inclusion of provenance strings in any YANG schema requiring them.

--- middle

# Introduction

OAM automation, generally based on closed-loop principles, requires at least two datasets to be used. Using the common terms in Control Theory, we need those from the plant (the network device or segment under control) and those to be used as reference (the desired values of the relevant data). The usual automation behavior compares these values and takes a decision, by whatever the method (algorithmic, rule-based, an AI model tuned by ML...) to decide on a control action according to this comparison. Assurance of the origin and integrity of these datasets, what we refer in this document as "provenance", becomes essential to guarantee a proper behavior of closed-loop automation.

When datasets are made available as an online data flow, provenance can be assessed by properties of the data transport protocol, as long as some kind of cryptographic protocol is used for source authentication, with TLS, SSH and IPsec as the main examples. But when these datasets are stored, go through some pre-processing or aggregation stages, or even cryptographic data transport is not available, provenance must be assessed by other means.

The original use case for this provenance mechanism is associated with {{YANGmanifest}}, in order to provide a proof of the origin and integrity of the provided metadata, and therefore the examples in this document use the modules described there, but it soon became clear that it could be extended to any YANG datamodel to support provenance evidence. An analysis of other potential use cases suggested the interest of defining an independent, generally applicable mechanism.

Provenance verification by signatures incorporated in the YANG data elements can be applied to any data processing pipeline, whether they rely on an online flow or use some kind of data store, such as data lakes or time-series databases. The application of recorded data for ML training or validation constitute the most relevant examples of these scenarios.

This document provides a mechanism for including digital signatures within YANG data. It applies COSE {{RFC9052}} to make the signature compact and reduce the resources required for calculating it. This mechanism is applicable to any serialization of the YANG data supporting a clear method for canonicalization, but this document considers three base ones: CBOR, JSON and XML.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

The term "data provenance" refers to a documented trail accounting for the origin of a piece of data and where it has moved from to where it is presently. The signature mechanism provided here can be recursively applied to allow this accounting for YANG data.

# Provenance Elements

To provide provenance for a YANG element, a new leaf element MUST be included, containing the COSE signature bitstring built according to the procedure defined in the following section. The provenance leaf MUST be of type provenance-signature, defined as follows:

~~~
typedef provenance-signature {
     type binary;
     description
      "The provenance-signature type represents a digital signature
       associated to the enclosing element. The signature is based
       on COSE and generated using a cannonicalized version of the
       enclosing element.";
     reference
      "RFC 9052: CBOR Object Signing and Encryption (COSE): Structures and Process
       draft-lopez-opsawg-yang-provenance";
}
~~~

When using it, a provenance-signature leaf MAY appear at any position in the enclosing element, but only one such leaf MUST be defined for the enclosing element. If the enclosing element contains other non-leaf elements, they MAY provide their own provenance-signature leaf, according to the same rule. In this case, the provenance-signature leaves in the children elements are applicable to the specific child element where they are enclosed, while the provenance-signature leaf enclosed in the top-most element is applicable to the whole element contents, including the children provenance-signature leaf themselves. This allows for recursive provenance validation, data aggregation, and the application of provenance verification of relevant children elements at different stages of any data processing pipeline.

As example, let us consider the two modules proposed in {{YANGmanifest}}. For the platform-manifest module, the provenance for a platform would be provided by the optional platform-provenance leaf shown below:

~~~
module: ietf-platform-manifest
  +--ro platforms
     +--ro platform* [id]
       +--ro platform-provenance?    provenance-signature
       +--ro id                      string
       +--ro name?                   string
       +--ro vendor?                 string
       +--ro vendor-pen?             uint32
       +--ro software-version?       string
       +--ro software-flavor?        string
       +--ro os-version?             string
       +--ro os-type?                string
       +--ro yang-push-streams
       |  +--ro stream* [name]
       |     +--ro name
       |     +--ro description?
       +--ro yang-library
       + . . .
       .
       .
       .
~~~

For data collections, the provenance of each one would be provided by the optional collector-provenance leaf, as shown below:

~~~
module: ietf-data-collection-manifest
  +--ro data-collections
     +--ro data-collection* [platform-id]
     +--ro platform-id
     |       -> /p-mf:platforms/platform/id
     +--ro collector-provenance?   provenance-signature
     +--ro yang-push-subscriptions
       +--ro subscription* [id]
         +--ro id
         |      sn:subscription-id
         +
         .
         .
         .
     + . . .
     |
     .
     .
     .
~~~

In both cases, and for the sake of brevity, the provenance element appears at the top of the enclosing element, but it is worth remarking they may appear anywhere within it.

Note that, in application of the recursion mechanism described above, a provenance element could be included at the top of any of the collections, supporting the verification of the provenance of the collection itself (as provided by a specific collector), without interfering with the verification of the provenance of each of the collection elements. As an example, in the case of the platform manifests it would look like:

~~~
module: ietf-platform-manifest
  +--ro platforms
     +--ro platform-collection-provenance? provenance-signature
     +--ro platform* [id]
       +--ro platform-provenance?          provenance-signature
       +--ro id                            string
       +--ro name?                         string
       +--ro vendor?                       string
       + . . .
       .
       .
       .
~~~

# Provenance Signature Strings

Signature strings are COSE single signature messages with \[nil\] payload, according to COSE conventions and registries, and with the following structure (as defined by {{RFC9052, Section 4.2}}):

~~~
COSE_Sign1 = [
protected /algorithm-identifier, kid, serialization-method/
unprotected /algorithm-parameters/
signature /using as external data the content
           of the (meta-)data without the signature leaf/
]
~~~

The COSE_Sign1 procedure yields a bitstring when building the signature and expects a bitstring for checking it, hence the proposed type for signature leaves. The structure of the COSE_Sign1 consists of:

* The algorithm-identifier, which MUST follow COSE conventions and registries.

* The kid (Key ID), to be locally agreed, used and interpreted by the signer and the signature validator. URIs {{RFC3986}} and RFC822-style {{RFC5322}} identifiers are typical values to be used as kid.

* The serialization-method, a string identifying the YANG serialization in use. It MUST be one of the three possible values "xml" (for XML serialization {{RFC7950}}), "json" (for JSON serialization {{RFC7951}}) or "cbor" (for CBOR serialization {{RFC9254}}).

* The value algorithm-parameters, which MUST follow the COSE conventions for providing relevant parameters to the signing algorithm.

* The signature for the enclosing element, to be produced and verified according to the procedure described below.

## Signature and Verification Procedures

To keep a concise signature and avoid the need for wrapping YANG constructs in COSE envelopes, the whole signature MUST be built and verified by means of externally supplied data, as defined in {{RFC9052, Section 4.3}}, with a \[nil\] payload.

The byte strings to be used as input to the signature and verification procedures MUST be built by:

* Taking the whole element enclosing the signature leaf.

* Eliminating the signature leaf element.

* Applying the corresponding canonicalization method as described in the following section.

## Canonicalization

Signature generation and verification require a canonicalization method to be applied, that depends on the serialization used. According to the three types of serialization defined, the following canonicalization methods MUST be applied:

* For CBOR, length-first core deterministic encoding, as defined by {{RFC8949}}.

* For JSON, JSON Canonicalization Scheme (JCS), as defined by {{RFC8785}}.

* For XML, Exclusive XML Canonicalization 1.0, as defined by {{XMLSig}}.

# YANG module

This module defines a provenance-signature type to be used in other YANG modules.

~~~
<CODE BEGINS> file "ietf-yang-provenance@2024-02-28.yang"
module ietf-yang-provenance {
  yang-version 1.1;
  namespace
    "urn:ietf:params:xml:ns:yang:ietf-yang-provenance";
  prefix iyangprov;

  organization "IETF OPSAWG (Operations and Management Area Working Group)";
  contact
    "WG Web:   <https://datatracker.ietf.org/wg/opsawg/>
     WG List:  <mailto:opsawg@ietf.org>

     Authors:  Alex Huang Feng
               <mailto:alex.huang-feng@insa-lyon.fr>
               Diego Lopez
               <mailto:diego.r.lopez@telefonica.com>
               Antonio Pastor
               <mailto:antonio.pastorperales@telefonica.com>";

  description
    "Defines a binary provenance-signature type to be used in other YANG
    modules.

    Copyright (c) 2024 IETF Trust and the persons identified as
    authors of the code.  All rights reserved.

    Redistribution and use in source and binary forms, with or without
    modification, is permitted pursuant to, and subject to the license
    terms contained in, the Revised BSD License set forth in Section
    4.c of the IETF Trust's Legal Provisions Relating to IETF Documents
    (https://trustee.ietf.org/license-info).

    This version of this YANG module is part of RFC XXXX; see the RFC
    itself for full legal notices.";

  revision 2024-02-28 {
    description
      "First revision";
    reference
      "RFC XXXX: Applying COSE Signatures for YANG Data Provenance";
  }

  typedef provenance-signature {
    type binary;
    description
      "The provenance-signature type represents a digital signature
      associated to the enclosing element. The signature is based
      on COSE and generated using a cannonicalized version of the
      enclosing element.";
    reference
      "RFC XXXX: Applying COSE Signatures for YANG Data Provenance";
  }
}
<CODE ENDS>
~~~

# Signature of NETCONF Event Notifications and YANG-Push Notifications

The signature mechanism proposed in this document can be used with NETCONF Event Notifications {{RFC5277}} and YANG-Push {{RFC8641}} to sign the generated notifications directly from the publisher nodes. The signature is added to the header of the Notification along with the eventTime leaf.

The following sections define the YANG module augmenting the ietf-notification module.


## YANG Tree Diagram

The following is the YANG tree diagram {{RFC8340}} for the ietf-notification-provenance augmentation within the ietf-notification.

~~~
module: ietf-notification-provenance

  augment-structure /inotif:notification:
    +-- notification-provenance?   iyangprov:provenance-signature
~~~

And the following is the full YANG tree diagram for the notification.

~~~
module: ietf-notification

  structure notification:
    +-- eventTime                             yang:date-and-time
    +-- inotifprov:notification-provenance?   iyangprov:provenance-signature
~~~

## YANG Module

The module augments ietf-notification module {{I-D.ahuang-netconf-notif-yang}} adding the signature leaf in the notification header.

~~~
<CODE BEGINS> file "ietf-notification-provenance@2024-02-28.yang"
module ietf-notification-provenance {
  yang-version 1.1;
  namespace
    "urn:ietf:params:xml:ns:yang:ietf-notification-provenance";
  prefix inotifprov;

  import ietf-notification {
    prefix inotif;
    reference
      "draft-ahuang-netconf-notif-yang: NETCONF Event Notification YANG";
  }
  import ietf-yang-provenance {
    prefix iyangprov;
    reference
      "RFC XXXX: Applying COSE Signatures for YANG Data Provenance";
  }
  import ietf-yang-structure-ext {
    prefix sx;
    reference
      "RFC 8791: YANG Data Structure Extensions";
  }

  organization "IETF OPSAWG (Operations and Management Area Working Group)";
  contact
    "WG Web:   <https://datatracker.ietf.org/wg/opsawg/>
     WG List:  <mailto:opsawg@ietf.org>

     Authors:  Alex Huang Feng
               <mailto:alex.huang-feng@insa-lyon.fr>
               Diego Lopez
               <mailto:diego.r.lopez@telefonica.com>
               Antonio Pastor
               <mailto:antonio.pastorperales@telefonica.com>";

  description
    "Defines a bynary provenance-signature type to be used in other YANG
    modules.

    Copyright (c) 2024 IETF Trust and the persons identified as
    authors of the code.  All rights reserved.

    Redistribution and use in source and binary forms, with or without
    modification, is permitted pursuant to, and subject to the license
    terms contained in, the Revised BSD License set forth in Section
    4.c of the IETF Trust's Legal Provisions Relating to IETF Documents
    (https://trustee.ietf.org/license-info).

    This version of this YANG module is part of RFC XXXX; see the RFC
    itself for full legal notices.";

  revision 2024-02-28 {
    description
      "First revision";
    reference
      "RFC XXXX: Applying COSE Signatures for YANG Data Provenance";
  }

  sx:augment-structure "/inotif:notification" {
    leaf notification-provenance {
      type iyangprov:provenance-signature;
      description
        "COSE signature of the content of the Notification for
        provenance verification.";
    }
  }
}
<CODE ENDS>
~~~

# Security Considerations

The provenance assessment mechanism described in this document relies on COSE {{RFC9052}} and the deterministic encoding or canonicalization procedures described by {{RFC8949}}, {{RFC8785}} and {{XMLSig}}. The security considerations made in these references are fully applicable here.

The verification step depends on the association of the kid (Key ID) with the proper public key. This is a local matter for the verifier and its specification is out of the scope of this document. The use of certificates, PKI mechanisms, or any other secure distribution of id-public key mappings is RECOMMENDED.


# IANA Considerations

## IETF XML Registry

This document registers the following URIs in the "IETF XML Registry" {{RFC3688}}:

~~~
  URI: urn:ietf:params:xml:ns:yang:ietf-yang-provenance
  Registrant Contact: The IESG.
  XML: N/A; the requested URI is an XML namespace.
~~~

~~~
  URI: urn:ietf:params:xml:ns:yang:ietf-notification-provenance
  Registrant Contact: The IESG.
  XML: N/A; the requested URI is an XML namespace.
~~~

## YANG Module Name

This document registers the following YANG modules in the "YANG Module Names" registry {{RFC6020}}:

~~~
  name: ietf-yang-provenance
  namespace: urn:ietf:params:xml:ns:yang:ietf-yang-provenance
  prefix: iyangprov
  reference: RFC XXXX
~~~

~~~
  name: ietf-notification-provenance
  namespace: urn:ietf:params:xml:ns:yang:ietf-notification-provenance
  prefix: inotifprov
  reference: RFC XXXX
~~~

Others?


--- back

# Acknowledgments
{:numbered="false"}
This document is based on work partially funded by the EU H2020 project SPIRS (grant 952622), and the EU Horizon Europe projects PRIVATEER (grant 101096110), HORSE (grant 101096342) and ACROSS (grant 101097122).
