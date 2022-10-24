%%%
title = "Selective Disclosure for JWTs (SD-JWT)"
abbrev = "SD-JWT"
ipr = "trust200902"
area = "Security"
workgroup = "Web Authorization Protocol"
keyword = ["security", "oauth2"]

[seriesInfo]
name = "Internet-Draft"
value = "draft-ietf-oauth-selective-disclosure-jwt-latest"
stream = "IETF"
status = "standard"

[[author]]
initials="D."
surname="Fett"
fullname="Daniel Fett"
organization="yes.com"
    [author.address]
    email = "mail@danielfett.de"
    uri = "https://danielfett.de/"


[[author]]
initials="K."
surname="Yasuda"
fullname="Kristina Yasuda"
organization="Microsoft"
    [author.address]
    email = "Kristina.Yasuda@microsoft.com"


%%%

.# Abstract

This document specifies conventions for creating JSON Web Token (JWT)
documents that support selective disclosure of JWT claim values.

{mainmatter}

# Introduction {#Introduction}

The JSON-based representation of claims in a signed JSON Web Token (JWT) [@!RFC7519] is
secured against modification using JSON Web Signature (JWS) [@!RFC7515] digital
signatures. A consumer of a signed JWT that has checked the
signature can safely assume that the contents of the token have not been
modified.  However, anyone receiving an unencrypted JWT can read all of the
claims and likewise, anyone with the decryption key receiving an encrypted JWT
can also read all of the claims.

This document describes a format for signed JWTs that supports selective
disclosure (SD-JWT), enabling sharing only a subset of the claims included in
the original signed JWT instead of releasing all the claims to every verifier.
During issuance, an SD-JWT is sent from the issuer to the holder alongside an
II-Disclosures object, a JSON object that contains the mapping
between raw claim values contained in the SD-JWT and the salts for each claim
value.

This document also defines a format for HS-Disclosures JWT, which convey
a subset of the claim values of an SD-JWT to the verifier. For presentation, the
holder creates a HS-Disclosures JWT and sends it together with the SD-JWT to the
verifier. To verify claim values received in HS-Disclosures JWT, the verifier uses the
salts values in the HS-Disclosures JWT to compute the digests of the claim values and
compare them to the ones in the SD-JWT.

One of the common use cases of a signed JWT is representing a user's identity
created by an issuer. As long as the signed JWT is one-time use, it typically
only contains those claims the user has consented to disclose to a specific
verifier. However, when a signed JWT is intended to be multi-use, it needs to
contain the superset of all claims the user might want to disclose to verifiers
at some point. The ability to selectively disclose a subset of these claims
depending on the verifier becomes crucial to ensure minimum disclosure and
prevent verifiers from obtaining claims irrelevant for the transaction at hand.

One example of such a multi-use JWT is a verifiable credential, a
tamper-evident credential with a cryptographically verifiable authorship that
contains claims about a subject. SD-JWTs defined in this document enable such
selective disclosure of claims.

While JWTs for claims describing natural persons are a common use case, the
mechanisms defined in this document can be used for many other use cases as
well.

This document also describes holder binding, or the concept of binding SD-JWT to
key material controlled by the subject of SD-JWT. Holder binding is optional to
implement.


## Conventions and Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL
NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED",
"MAY", and "OPTIONAL" in this document are to be interpreted as
described in BCP 14 [@!RFC2119] [@!RFC8174] when, and only when, they
appear in all capitals, as shown here.

**base64url** denotes the URL-safe base64 encoding without padding defined in
Section 2 of [@!RFC7515].

# Terms and Definitions

Selectively Disclosable JWT (SD-JWT)
:  A JWT [@!RFC7515] created by the issuer, which is signed as a JWS [@!RFC7515],
   that supports selective disclosure as defined in this document.

Disclosure
: A combination of the cleartext claim value, the cleartext claim name, a salt and
   optionally blinded claim name value that is used to calculate a digest for a certain claim.

Issuer-Issued Disclosures Object (II-Disclosures Object)
:  A JSON object created by the issuer that contains mapping between
   raw claim values contained in the SD-JWT and the salts for each claim value.

Holder-Selected Disclosures JWT (HS-Disclosures JWT)
:  A JWT created by the Holder that contains the Disclosures from an Issuer-Issued Disclosures Object that the Holder is disclosing to the Verifier. In addition to the Disclosures, it can contain other properties and may be signed by the Holder.

claim values of an SD-JWT in a verifiable way.

Holder binding
:  Ability of the holder to prove legitimate possession of SD-JWT by proving
   control over the same private key during the issuance and presentation. SD-JWT signed by the issuer contains
   a public key or a reference to a public key that matches to the private key controlled by the holder.

Claim name blinding
:  Feature that enables to blind not only claim values, but also claim names of the claims
that are included in SD-JWT but are not disclosed to the verifier in the HS-Disclosures JWT.

Issuer
:  An entity that creates SD-JWTs.

Holder
:  An entity that received SD-JWTs from the issuer and has control over them.

Verifier
:  An entity that requests, checks and extracts the claims from HS-Disclosures JWT.

Selective disclosure
: Process of a Holder disclosing to a Verifier a subset of claims contained in a claim set issued by an Issuer.

Note: discuss if we want to include Client, Authorization Server for the purpose of
ensuring continuity and separating the entity from the actor.

# Flow Diagram

~~~ ascii-art
           +------------+
           |            |
           |   Issuer   |
           |            |
           +------------+
                 |
               Issues
           SD-JWT and Issuer-Issued Disclosures Object
                 |
                 v
           +------------+
           |            |
           |   Holder   |
           |            |
           +------------+
                 |
              Presents
         Holder-Selected Disclosures JWT and SD-JWT
                 |
                 v
           +-------------+
           |             |+
           |  Verifiers  ||+
           |             |||
           +-------------+||
            +-------------+|
             +-------------+
~~~
Figure: SD-JWT Issuance and Presentation Flow

# Concepts

In the following, the contents of SD-JWTs and HS-Disclosures JWTs are described at a
conceptual level, abstracting from the data formats described afterwards.

## Creating an SD-JWT

An SD-JWT, at its core, is a digitally signed document containing digests over the claim values with random salts and other metadata.
It MUST be digitally signed using the issuer's private key.

```
SD-JWT-DOC = (METADATA, SD-CLAIMS)
SD-JWT = SD-JWT-DOC | SIG(SD-JWT-DOC, ISSUER-PRIV-KEY)
```

`SD-CLAIMS` is an object with claim names (`CLAIM-NAME`) mapped to the digests over the claim values (`CLAIM-VALUE`) with random salts (`SALT`). Digests are calculated using a digest derivation function such as a hash function, HMAC, or other (`DIGEST-DERIVATION()`):

```
SD-CLAIMS = (
    CLAIM-NAME: DIGEST-DERIVATION(SALT, CLAIM-VALUE)
)*
```

When an HMAC or another type of derivation function is used for digest calculation, a secret cryptographic key or other cryptographic secret is used instead of a salt value.
However, the term "salt" is used throughout this document for brevity.

`SD-CLAIMS` can also be nested deeper to capture more complex objects, as will be shown later.

`SD-JWT` is sent from the issuer to the holder, together with the mapping of the plain-text claim values, the salt values, and potentially some other information.

## Creating an Holder-Selected Disclosures JWT

To disclose to a verifier a subset of the SD-JWT claim values, a holder creates a JWT such as the
following:

```
HOLDER-SELECTED-DISCLOSURES-DOC = (METADATA, SD-DISCLOSURES)
HOLDER-SELECTED-DISCLOSURES-JWT = HOLDER-SELECTED-DISCLOSURES-DOC
```


`SD-DISCLOSURES` follows the structure of `SD-CLAIMS` and can be a simple object with claim names mapped to values and salts:

```
SD-DISCLOSURES = (
    CLAIM-NAME: (DISCLOSED-SALT, DISCLOSED-VALUE)
)
```

Just as `SD-CLAIMS`, `SD-DISCLOSURES` can be more complex as well.

`HOLDER-SELECTED-DISCLOSURES-JWT` is sent together with `SD-JWT` from the holder to the
verifier.

## Optional Holder Binding

Some use-cases may require holder binding.

If holder binding is desired, `SD-JWT` must contain information about key material controlled by the holder:

```
SD-JWT-DOC = (METADATA, HOLDER-PUBLIC-KEY, SD-CLAIMS)
```

Note: How the public key is included in SD-JWT is out of scope of this document. It can be passed by value or by reference.

With holder binding, the `HOLDER-SELECTED-DISCLOSURES-JWT` is signed by the holder using its private key. It therefore looks as follows:

```
HOLDER-SELECTED-DISCLOSURES = HOLDER-SELECTED-DISCLOSURES-DOC | SIG(HOLDER-SELECTED-DISCLOSURES-DOC, HOLDER-PRIV-KEY)
```

### Optional Claim Name Blinding

If claim name blinding is used, `SD-CLAIMS` is created as follows:
```
SD-CLAIMS = (
    CLAIM-NAME-PLACEHOLDER: DIGEST-DERIVATION(SALT, CLAIM-VALUE, CLAIM-NAME)
)*
```

`CLAIM-NAME-PLACEHOLDER` is a placeholder used instead of the original claim
name, chosen such that it does not leak information about the claim name (e.g.,
randomly).

The contents of `SD-DISCLOSURES` are modified as follows:
```
SD-DISCLOSURES = (
    CLAIM-NAME-PLACEHOLDER: (DISCLOSED-SALT, DISCLOSED-VALUE, DISCLOSED-CLAIM-NAME)
)
```
Note that blinded and unblinded claim names can be mixed in `SD-CLAIMS` and accordingly in `SD-DISCLOSURES`.

## Verifying an Holder-Selected Disclosures JWT

A verifier checks that

 * for each claim in `HOLDER-SELECTED-DISCLOSURES`, the digest over the disclosed values
   matches the digest under the given claim name in `SD-JWT`,
 * if holder binding is used, the `HOLDER-SELECTED-DISCLOSURES` was signed by the private key
 belonging to `HOLDER-PUBLIC-KEY`.

The detailed algorithm is described below.

# Data Formats

This section defines data formats for SD-JWT (containing digests of the salted
claim values), Issuer-Issued Disclosures (containing the mapping of the
plain-text claim values and the salt values), and HS-Disclosures
(containing a subset of the same mapping).

## The Challenge of Canonicalization

When receiving an SD-JWT with an associated Release, a verifier must be able to
re-compute digests of the disclosed claim value and, given the same input values,
obtain the same digest values as signed by the issuer.

Usually, JSON-based formats transport claim values as simple properties of a JSON object such as this:

```
...
  "family_name": "Möbius",
  "address": {
    "street_address": "Schulstr. 12",
    "locality": "Schulpforta"
  }
...
```

However, a problem arises when computation over the data need to be performed and verified, like signing or computing digests. Common signature schemes require the same byte string as input to the
signature verification as was used for creating the signature. In the digest derivation approach outlined above, the same problem exists: for the issuer and the
verifier to arrive at the same digest, the same byte string must be hashed.

JSON, however, does not prescribe a unique encoding for data, but allows for variations in the encoded string. The data above, for example, can be encoded as

```
...
"family_name": "M\u00f6bius",
"address": {
  "street_address": "Schulstr. 12",
  "locality": "Schulpforta"
}
...
```

or as

```
...
"family_name": "Möbius",
"address": {"locality":"Schulpforta", "street_address":"Schulstr. 12"}
...
```

The two representations `"M\u00f6bius"` and `"Möbius"` are very different on the byte-level, but yield
equivalent objects. Same for the representations of `address`, varying in white space and order of elements in the object.

The variations in white space, ordering of object properties, and encoding of
Unicode characters are all allowed by the JSON specification. Other variations,
e.g., concerning floating-point numbers, are described in [@RFC8785]. Variations
can be introduced whenever JSON data is serialized or deserialized and unless
dealt with, will lead to different digests and the inability to verify
signatures.

There are generally two approaches to deal with this problem:

1. Canonicalization: The data is transferred in JSON format, potentially
   introducing variations in its representation, but is transformed into a
   canonical form before computing a digest. Both the issuer and the verifier
   must use the same canonicalization algorithm to arrive at the same byte
   string for computing a digest.
2. Source string encoding: Instead of transferring data in JSON format that may
   introduce variations, the serialized data that is used as the digest input is
   transferred from the issuer to the verifier. This means that the verifier can
   easily check the digest over the byte string before deserializing the data.

Mixed approaches are conceivable, i.e., transferring both the original JSON data
plus a string suitable for computing a digest, but such approaches can easily lead to
undetected inconsistencies resulting in time-of-check-time-of-use type security
vulnerabilities.

In this specification, the source string encoding approach is used, as it allows
for simple and reliable interoperability without the requirement for a
canonicalization library. To encode the source string, JSON itself is used. This
approach means that SD-JWTs can be implemented purely based on widely available
JSON encoding and decoding libraries without the need for a custom data format
for encoding data.

To produce a source string to compute a digest, the data is put into a JSON object
together with the salt value, like so (non-normative example, see
(#sd_digests_claim) for details):

```
{"s": "6qMQvRL5haj", "v": "Möbius"}
```

Or, for the address example above:
```
{"s": "al1N3Zom221", "v":
  {"locality": "Schulpforta", "street_address": "Schulstr. 12"}}
```
(Line break and indentation of the second line for presentation only!)

This object is then JSON-encoded and used as the source string. The JSON-encoded value is transferred in the HS-Disclosures instead of the original JSON data:

```
"family_name": "{\"s\": \"6qMQvRL5haj\", \"v\": \"M\\u00f6bius\"}"
```

Or, for the address example:
```
"address": "{\"s\": \"al1N3Zom221\", \"v\":
  {\"locality\": \"Schulpforta\",
  \"street_address\": \"Schulstr. 12\"}}"
```
(Line break and indentation of the second and third line for presentation only!)

A verifier can then easily check the digest over the source string before
extracting the original JSON data. Variations in the encoding of the source
string are implicitly tolerated by the verifier, as the digest is computed over a
predefined byte string and not over a JSON object.

Since the encoding is based on JSON, all value types that are allowed in JSON
are also allowed in the `v` property in the source string. This includes
numbers, strings, booleans, arrays, and objects.

It is important to note that the HS-Disclosures containing the source string is
neither intended nor suitable for direct consumption by an application that
needs to access the disclosed claim values. The SD-JWT-Release are only intended
to be used by a verifier to check the digests over the source strings and to extract
the original JSON data. The original JSON data is then used by the application.
See (#processing_model) for details.

## Format of an SD-JWT

An SD-JWT is a JWT that MUST be signed using the issuer's private key. The
payload of an SD-JWT MUST contain the `sd_digests` and `sd_digest_derivation_alg` claims
described in the following, and MAY contain a holder's public key or a reference
thereto, as well as further claims such as `iss`, `iat`, etc. as defined or
required by the application using SD-JWTs.

### `sd_digests` Claim (Digests of Selectively Disclosable Claims) {#sd_digests_claim}

The property `sd_digests` MUST be used by the issuer to include digests of the salted claim values for any claim that is intended to be selectively disclosable.

The issuer MUST choose a random salt value for each claim. It is
RECOMMENDED to do so by base64url-encoding a cryptographically secure
nonce. See (#salt-minlength) for further requirements.

The issuer MUST generate the digests over a JSON literal according to
[@!RFC8259] that is formed by
JSON-encoding an object with the following contents:

 * REQUIRED with the key `s`: the salt value,
 * REQUIRED with the key `v`: the claim value (either a string or a more complex object, e.g., for the [@OIDC] `address` claim),
 * OPTIONAL, with the key `n`: the claim name (if claim name blinding is to be used for this claim).

The following is an example for a JSON literal without claim name blinding:

```
{"s": "6qMQvRL5haj", "v": "Peter"}
```

The following is an example for a JSON literal with claim name blinding:

```
{"s": "6qMQvRL5haj", "v": "Peter", "n": "given_name"}
```

The `sd_digests` claim contains an object where claim names are mapped
to the respective digests. If a claim name is to be blinded, the digests
MUST contain the `n` key as described above and the claim name in
`sd_digests` MUST be replaced by a placeholder name that does not leak
information about the claim's original name. The same placeholder name
will be used in the II-Disclosures (`sd_ii_disclosures`) and
HS-Disclosures (`sd_hs_disclosures`) described below.

To this end, the issuer MUST choose a random placeholder name for each
claim that is to be blinded. It is RECOMMENDED to do so by
base64url-encoding a cryptographically secure nonce. See
(#blinding-claim-names) for further requirements.

#### Flat and Structured `sd_digests` objects

The `sd_digests` object can be a 'flat' object, directly containing all claim
names and digests without any deeper structure. The `sd_digests`
object can also be a 'structured' object, where some claims and their respective
digests are contained in places deeper in the structure. It is at the issuer's
discretion whether to use a 'flat' or 'structured' `sd_digests` SD-JWT object,
and how to structure it such that it is suitable for the use case.

Example 1 below is a non-normative example of an SD-JWT using a 'flat'
`sd_digests` object and Example 2a in the appendix shows a non-normative example
of an SD-JWT using a 'structured' `sd_digests` object. The difference between
the examples is how the `address` claim is disclosed.

Appendix 2 shows a more complex example using claims from eKYC (todo:
reference).

### Digest Derivation Function Claim

The claim `sd_digest_derivation_alg` indicates the digest derivation algorithm
used by the Issuer to generate the digests over the salts and the
claim values.

The digest derivation algorithm identifier MUST be one of the following:
- a hash algorithm value from the "Hash Name String" column in the IANA "Named Information Hash Algorithm" registry [IANA.Hash.Algorithms]
- an HMAC algorithm value from the "Algorithmn Name" column in the IANA "JSON Web Signature and Encryption Algorithms" registry [IANA.JWS.Algorithms]
- a value defined in another specification and/or profile of this specification

To promote interoperability, implementations MUST support the SHA-256 hash algorithm.


See (#security_considerations) for requirements regarding entropy of the salt, minimum length of the salt, and choice of a digest derivation algorithm.

### Holder Public Key Claim

If the issuer wants to enable holder binding, it MAY include a public key
associated with the holder, or a reference thereto.

It is out of the scope of this document to describe how the holder key pair is
established. For example, the holder MAY provide a key pair to the issuer,
the issuer MAY create the key pair for the holder, or
holder and issuer MAY use pre-established key material.

Note: Examples in this document use `cnf` Claim defined in [@RFC7800] to include raw public key by value in SD-JWT.

## Example 1: SD-JWT

This example and Example 2a in the appendix use the following object as the set
of claims that the Issuer is issuing:

{#example-simple-user_claims}
```json
{
  "sub": "6c5c0a49-b589-431d-bae7-219122a9ec2c",
  "given_name": "John",
  "family_name": "Doe",
  "email": "johndoe@example.com",
  "phone_number": "+1-202-555-0101",
  "address": {
    "street_address": "123 Main St",
    "locality": "Anytown",
    "region": "Anystate",
    "country": "US"
  },
  "birthdate": "1940-01-01"
}
```

The following non-normative example shows the payload of an SD-JWT. The issuer
is using a flat structure, i.e., all of the claims the `address` claim can only
be disclosed in full.

{#example-simple-sd_jwt_payload}
```json
{
  "iss": "https://example.com/issuer",
  "cnf": {
    "jwk": {
      "kty": "RSA",
      "n": "pm4bOHBg-oYhAyPWzR56AWX3rUIXp11_ICDkGgS6W3ZWLts-hzwI3x656
        59kg4hVo9dbGoCJE3ZGF_eaetE30UhBUEgpGwrDrQiJ9zqprmcFfr3qvvkGjt
        th8Zgl1eM2bJcOwE7PCBHWTKWYs152R7g6Jg2OVph-a8rq-q79MhKG5QoW_mT
        z10QT_6H4c7PjWG1fjh8hpWNnbP_pv6d1zSwZfc5fl6yVRL0DV0V3lGHKe2Wq
        f_eNGjBrBLVklDTk8-stX_MWLcR-EGmXAOv0UBWitS_dXJKJu-vXJyw14nHSG
        uxTIK2hx1pttMft9CsvqimXKeDTU14qQL1eE7ihcw",
      "e": "AQAB"
    }
  },
  "iat": 1516239022,
  "exp": 1516247022,
  "sd_digest_derivation_alg": "sha-256",
  "sd_digests": {
    "sub": "OMdwkk2HPuiInPypWUWMxot1Y2tStGsLuIcDMjKdXMU",
    "given_name": "AfKKH4a0IZki8MFDythFaFS_Xqzn-wRvAMfiy_VjYpE",
    "family_name": "eUmXmry32JiK_76xMasagkAQQsmSVdW57Ajk18riSF0",
    "email": "-Rcr4fDyjwlM_itcMxoQZCE1QAEwyLJcibEpH114KiE",
    "phone_number": "Jv2nw0C1wP5ASutYNAxrWEnaDRIpiF0eTUAkUOp8F6Y",
    "address": "ZrjKs-RmEAVeAYSzSw6GPFrMpcgctCfaJ6t9qQhbfJ4",
    "birthdate": "qXPRRPdpNaebP8jtbEpO-skF4n7v7ASTh8oLg0mkAdQ"
  }
}
```

Important: Throughout the examples in this document, line breaks had to
be added to JSON strings and base64-encoded strings (as shown in the
next example) to adhere to the 72 character limit for lines in RFCs and
for readability. JSON does not allow line breaks in strings.

The SD-JWT is then signed by the issuer to create a document like the following:

{#example-simple-serialized_sd_jwt}
```
eyJhbGciOiAiUlMyNTYiLCAia2lkIjogImNBRUlVcUowY21MekQxa3pHemhlaUJhZzBZU
kF6VmRsZnhOMjgwTmdIYUEifQ.eyJpc3MiOiAiaHR0cHM6Ly9leGFtcGxlLmNvbS9pc3N
1ZXIiLCAiY25mIjogeyJqd2siOiB7Imt0eSI6ICJSU0EiLCAibiI6ICJwbTRiT0hCZy1v
WWhBeVBXelI1NkFXWDNyVUlYcDExX0lDRGtHZ1M2VzNaV0x0cy1oendJM3g2NTY1OWtnN
GhWbzlkYkdvQ0pFM1pHRl9lYWV0RTMwVWhCVUVncEd3ckRyUWlKOXpxcHJtY0ZmcjNxdn
ZrR2p0dGg4WmdsMWVNMmJKY093RTdQQ0JIV1RLV1lzMTUyUjdnNkpnMk9WcGgtYThycS1
xNzlNaEtHNVFvV19tVHoxMFFUXzZINGM3UGpXRzFmamg4aHBXTm5iUF9wdjZkMXpTd1pm
YzVmbDZ5VlJMMERWMFYzbEdIS2UyV3FmX2VOR2pCckJMVmtsRFRrOC1zdFhfTVdMY1ItR
UdtWEFPdjBVQldpdFNfZFhKS0p1LXZYSnl3MTRuSFNHdXhUSUsyaHgxcHR0TWZ0OUNzdn
FpbVhLZURUVTE0cVFMMWVFN2loY3ciLCAiZSI6ICJBUUFCIn19LCAiaWF0IjogMTUxNjI
zOTAyMiwgImV4cCI6IDE1MTYyNDcwMjIsICJzZF9kaWdlc3RfZGVyaXZhdGlvbl9hbGci
OiAic2hhLTI1NiIsICJzZF9kaWdlc3RzIjogeyJzdWIiOiAiT01kd2trMkhQdWlJblB5c
FdVV014b3QxWTJ0U3RHc0x1SWNETWpLZFhNVSIsICJnaXZlbl9uYW1lIjogIkFmS0tING
EwSVpraThNRkR5dGhGYUZTX1hxem4td1J2QU1maXlfVmpZcEUiLCAiZmFtaWx5X25hbWU
iOiAiZVVtWG1yeTMySmlLXzc2eE1hc2Fna0FRUXNtU1ZkVzU3QWprMThyaVNGMCIsICJl
bWFpbCI6ICItUmNyNGZEeWp3bE1faXRjTXhvUVpDRTFRQUV3eUxKY2liRXBIMTE0S2lFI
iwgInBob25lX251bWJlciI6ICJKdjJudzBDMXdQNUFTdXRZTkF4cldFbmFEUklwaUYwZV
RVQWtVT3A4RjZZIiwgImFkZHJlc3MiOiAiWnJqS3MtUm1FQVZlQVlTelN3NkdQRnJNcGN
nY3RDZmFKNnQ5cVFoYmZKNCIsICJiaXJ0aGRhdGUiOiAicVhQUlJQZHBOYWViUDhqdGJF
cE8tc2tGNG43djdBU1RoOG9MZzBta0FkUSJ9fQ.cAMV58Un7veKm2tlSgUHsobwYbM7Y9
fzHrTSEXEj7Coq-tUwpb9t1JG6IirrCI_ogsIHa_0yLhxFWiaWZfcAmHh9_luJvryhjag
ZdD0z5SAb-2yq4HJQaBsLXliVcFEFt1I8UCJrnpETwQzPohhyB4Gjkupz1pfxtSAxsIEZ
G8fk5N1yC9rUvWHanzyrQMOmTxiNQHuSvMqyOlaC5Cszlkj58SQhNx88SmB7xAmCgdZsd
L8MipmOq_U5wmrIna8vzIyJpPuP0FQnARmtRC7CEjs4RKH4UF2Du2pyyXGSfQ2drWMp9j
HcljwQrmgZ0sNpFnvH8tA_W02ykNO_uoi2BA
```


## Format of an Issuer-Issued Disclosures Object

Besides the SD-JWT itself, the holder needs to learn the raw claim values that
are contained in the SD-JWT, along with the precise input to the digest
calculation, and the salts. There MAY be other information the issuer needs to
communicate to the holder, such as a private key if the issuer selected the
holder key pair.

An Issuer-Issued Disclosures Object (II-Disclosures Object) is a JSON object containing at least the
top-level property `sd_ii_disclosures`. Its structure mirrors the one of `sd_digests` in
the SD-JWT, but the values are the inputs to the digest calculations the issuer
used, as strings.

The II-Disclosures Object MAY contain further properties, for example, to transport the holder
private key.

## Example: Issuer-Issued Disclosures Object for the Flat SD-JWT in Example 1

The II-Disclosures Object for Example 1 is as follows:

{#example-simple-svc_payload}
```json
{
  "sd_ii_disclosures": {
    "sub": "{\"s\": \"2GLC42sKQveCfGfryNRN9w\", \"v\":
      \"6c5c0a49-b589-431d-bae7-219122a9ec2c\"}",
    "given_name": "{\"s\": \"6Ij7tM-a5iVPGboS5tmvVA\", \"v\":
      \"John\"}",
    "family_name": "{\"s\": \"Qg_O64zqAxe412a108iroA\", \"v\":
      \"Doe\"}",
    "email": "{\"s\": \"Pc33JM2LchcU_lHggv_ufQ\", \"v\":
      \"johndoe@example.com\"}",
    "phone_number": "{\"s\": \"lklxF5jMYlGTPUovMNIvCA\", \"v\":
      \"+1-202-555-0101\"}",
    "address": "{\"s\": \"5bPs1IquZNa0hkaFzzzZNw\", \"v\":
      {\"street_address\": \"123 Main St\", \"locality\":
      \"Anytown\", \"region\": \"Anystate\", \"country\": \"US\"}}",
    "birthdate": "{\"s\": \"y1sVU5wdfJahVdgwPgS7RQ\", \"v\":
      \"1940-01-01\"}"
  }
}
```

Important: As described above, digests are calculated over the JSON literal
formed by serializing an object containing the salt, the claim value, and
optionally the claim name. This ensures that issuer and verifier use the same
input to their digest derivation algorithms and avoids issues with canonicalization of JSON
values that would lead to different digests. The II-Disclosures Object therefore maps claim
names to JSON-encoded arrays.

## Sending SD-JWT and Issuer-Issued Disclosures Object during Issuance

For transporting the II-Disclosures Object together with the SD-JWT from the issuer to the holder,
the II-Disclosures Object is base64url-encoded and appended to the SD-JWT using a period character `.` as the
separator.

The II-Disclosures Object and SD-JWT are implicitly linked through the digest values of the claims
in the II-Disclosures Object that is included in the SD-JWT. To ensure that the correct II-Disclosures Object and
SD-JWT pairings are being used, the holder SHOULD verify the binding between
II-Disclosures Object and SD-JWT as defined in the Verification Section of this document.

For Example 1, the combined format looks as follows:

{#example-simple-combined_sd_jwt_svc}
```
eyJhbGciOiAiUlMyNTYiLCAia2lkIjogImNBRUlVcUowY21MekQxa3pHemhlaUJhZzBZU
kF6VmRsZnhOMjgwTmdIYUEifQ.eyJpc3MiOiAiaHR0cHM6Ly9leGFtcGxlLmNvbS9pc3N
1ZXIiLCAiY25mIjogeyJqd2siOiB7Imt0eSI6ICJSU0EiLCAibiI6ICJwbTRiT0hCZy1v
WWhBeVBXelI1NkFXWDNyVUlYcDExX0lDRGtHZ1M2VzNaV0x0cy1oendJM3g2NTY1OWtnN
GhWbzlkYkdvQ0pFM1pHRl9lYWV0RTMwVWhCVUVncEd3ckRyUWlKOXpxcHJtY0ZmcjNxdn
ZrR2p0dGg4WmdsMWVNMmJKY093RTdQQ0JIV1RLV1lzMTUyUjdnNkpnMk9WcGgtYThycS1
xNzlNaEtHNVFvV19tVHoxMFFUXzZINGM3UGpXRzFmamg4aHBXTm5iUF9wdjZkMXpTd1pm
YzVmbDZ5VlJMMERWMFYzbEdIS2UyV3FmX2VOR2pCckJMVmtsRFRrOC1zdFhfTVdMY1ItR
UdtWEFPdjBVQldpdFNfZFhKS0p1LXZYSnl3MTRuSFNHdXhUSUsyaHgxcHR0TWZ0OUNzdn
FpbVhLZURUVTE0cVFMMWVFN2loY3ciLCAiZSI6ICJBUUFCIn19LCAiaWF0IjogMTUxNjI
zOTAyMiwgImV4cCI6IDE1MTYyNDcwMjIsICJzZF9oYXNoX2FsZyI6ICJzaGEtMjU2Iiwg
InNkX2RpZ2VzdHMiOiB7InN1YiI6ICJPTWR3a2sySFB1aUluUHlwV1VXTXhvdDFZMnRTd
EdzTHVJY0RNaktkWE1VIiwgImdpdmVuX25hbWUiOiAiQWZLS0g0YTBJWmtpOE1GRHl0aE
ZhRlNfWHF6bi13UnZBTWZpeV9WallwRSIsICJmYW1pbHlfbmFtZSI6ICJlVW1YbXJ5MzJ
KaUtfNzZ4TWFzYWdrQVFRc21TVmRXNTdBamsxOHJpU0YwIiwgImVtYWlsIjogIi1SY3I0
ZkR5andsTV9pdGNNeG9RWkNFMVFBRXd5TEpjaWJFcEgxMTRLaUUiLCAicGhvbmVfbnVtY
mVyIjogIkp2Mm53MEMxd1A1QVN1dFlOQXhyV0VuYURSSXBpRjBlVFVBa1VPcDhGNlkiLC
AiYWRkcmVzcyI6ICJacmpLcy1SbUVBVmVBWVN6U3c2R1BGck1wY2djdENmYUo2dDlxUWh
iZko0IiwgImJpcnRoZGF0ZSI6ICJxWFBSUlBkcE5hZWJQOGp0YkVwTy1za0Y0bjd2N0FT
VGg4b0xnMG1rQWRRIn19.QgoJn9wkjFvM9bAr0hTDHLspuqdA21WzfBRVHkASa2ck4PFD
3TC9MiZSi3AiRytRbYT4ZzvkH3BSbm6vy68y62gj0A6OYvZ1Z60Wxho14bxZQveJZgw3u
_lMvYj6GKiUtskypFEHU-Kd-LoDVqEpf6lPQHdpsac__yQ_JL24oCEBlVQRXB-T-6ZNZf
ID6JafSkNNCYQbI8nXbzIEp1LBFm0fE8eUd4G4yPYOj1SeuR6Gy92T0vAoL5QtpIAHo49
oAmiSIj6DQNl2cNYs74jhrBIcNZyt4l8H1lV20wS5OS3T0vXaYD13fgm0p4iWD9cVg3HK
ShUVulEyrSbq94jIKg.eyJzZF9yZWxlYXNlIjogeyJzdWIiOiAie1wic1wiOiBcIjJHTE
M0MnNLUXZlQ2ZHZnJ5TlJOOXdcIiwgXCJ2XCI6IFwiNmM1YzBhNDktYjU4OS00MzFkLWJ
hZTctMjE5MTIyYTllYzJjXCJ9IiwgImdpdmVuX25hbWUiOiAie1wic1wiOiBcIjZJajd0
TS1hNWlWUEdib1M1dG12VkFcIiwgXCJ2XCI6IFwiSm9oblwifSIsICJmYW1pbHlfbmFtZ
SI6ICJ7XCJzXCI6IFwiUWdfTzY0enFBeGU0MTJhMTA4aXJvQVwiLCBcInZcIjogXCJEb2
VcIn0iLCAiZW1haWwiOiAie1wic1wiOiBcIlBjMzNKTTJMY2hjVV9sSGdndl91ZlFcIiw
gXCJ2XCI6IFwiam9obmRvZUBleGFtcGxlLmNvbVwifSIsICJwaG9uZV9udW1iZXIiOiAi
e1wic1wiOiBcImxrbHhGNWpNWWxHVFBVb3ZNTkl2Q0FcIiwgXCJ2XCI6IFwiKzEtMjAyL
TU1NS0wMTAxXCJ9IiwgImFkZHJlc3MiOiAie1wic1wiOiBcIjViUHMxSXF1Wk5hMGhrYU
Z6enpaTndcIiwgXCJ2XCI6IHtcInN0cmVldF9hZGRyZXNzXCI6IFwiMTIzIE1haW4gU3R
cIiwgXCJsb2NhbGl0eVwiOiBcIkFueXRvd25cIiwgXCJyZWdpb25cIjogXCJBbnlzdGF0
ZVwiLCBcImNvdW50cnlcIjogXCJVU1wifX0iLCAiYmlydGhkYXRlIjogIntcInNcIjogX
CJ5MXNWVTV3ZGZKYWhWZGd3UGdTN1JRXCIsIFwidlwiOiBcIjE5NDAtMDEtMDFcIn0ifX
0
```

(Line breaks for presentation only.)

## Format of an Holder-Selected Disclosures JWT

HS-Disclosures JWT contains claim values and the salts of the claims that the holder
has consented to disclose to the Verifier. This enables the Verifier to verify
the claims received from the holder by computing the digests of the claim
values and the salts revealed in the HS-Disclosures JWT using the digest derivation algorithm
specified in SD-JWT and comparing them to the digests included in SD-JWT.

For each claim, an array of the salt and the claim value is contained in the
`sd_hs_disclosures` object. The structure of an `sd_hs_disclosures` object in the HS-Disclosures JWT is the same as the structure of an `sd_ii_disclosures` object in II-Disclosures Object.

The HS-Disclosures JWT MAY contain further claims, for example, to ensure a binding
to a concrete transaction (in the example the `nonce` and `aud` claims).

When the holder sends the HS-Disclosures JWT to the Verifier, the HS-Disclosures JWT MUST be a JWS
represented as the JWS Compact Serialization as described in
Section 7.1 of [@!RFC7515].

If holder binding is desired, the HS-Disclosures JWT is signed by the holder. If no
holder binding is to be used, the `none` algorithm is used, i.e., the document
is not signed. TODO: Change to plain base64 to avoid alg=none issues

## Example: Holder-Selected Disclosures JWT for Example 1

The following is a non-normative example of the contents of a HS-Disclosures JWT for Example 1:

{#example-simple-sd_jwt_release_payload}
```json
{
  "nonce": "XZOUco1u_gEPknxS78sWWg",
  "aud": "https://example.com/verifier",
  "sd_hs_disclosures": {
    "given_name": "{\"s\": \"6Ij7tM-a5iVPGboS5tmvVA\", \"v\":
      \"John\"}",
    "family_name": "{\"s\": \"Qg_O64zqAxe412a108iroA\", \"v\":
      \"Doe\"}",
    "address": "{\"s\": \"5bPs1IquZNa0hkaFzzzZNw\", \"v\":
      {\"street_address\": \"123 Main St\", \"locality\":
      \"Anytown\", \"region\": \"Anystate\", \"country\": \"US\"}}"
  }
}
```

For each claim, a JSON literal that decodes to an object with the and the claim
value (plus optionally the claim name) is contained in the `sd_hs_disclosures` object.

Again, the HS-Disclosures JWT follows the same structure as the `sd_digests` in the SD-JWT.

Below is a non-normative example of a representation of the HS-Disclosures JWT using JWS Compact
Serialization:

{#example-simple-serialized_sd_jwt_release}
```
eyJhbGciOiAiUlMyNTYiLCAia2lkIjogIkxkeVRYd0F5ZnJpcjRfVjZORzFSYzEwVThKZ
ExZVHJFQktKaF9oNWlfclUifQ.eyJub25jZSI6ICJYWk9VY28xdV9nRVBrbnhTNzhzV1d
nIiwgImF1ZCI6ICJodHRwczovL2V4YW1wbGUuY29tL3ZlcmlmaWVyIiwgInNkX3JlbGVh
c2UiOiB7ImdpdmVuX25hbWUiOiAie1wic1wiOiBcIjZJajd0TS1hNWlWUEdib1M1dG12V
kFcIiwgXCJ2XCI6IFwiSm9oblwifSIsICJmYW1pbHlfbmFtZSI6ICJ7XCJzXCI6IFwiUW
dfTzY0enFBeGU0MTJhMTA4aXJvQVwiLCBcInZcIjogXCJEb2VcIn0iLCAiYWRkcmVzcyI
6ICJ7XCJzXCI6IFwiNWJQczFJcXVaTmEwaGthRnp6elpOd1wiLCBcInZcIjoge1wic3Ry
ZWV0X2FkZHJlc3NcIjogXCIxMjMgTWFpbiBTdFwiLCBcImxvY2FsaXR5XCI6IFwiQW55d
G93blwiLCBcInJlZ2lvblwiOiBcIkFueXN0YXRlXCIsIFwiY291bnRyeVwiOiBcIlVTXC
J9fSJ9fQ.fw4xRl7m1mDPCZvCTn3GOr2PgBZ--fTKfy7s-GuEifNvzW5KsJaBBFvzdZzt
m25XGhk29uw-XwEw00r0hyxXLBvWfA0XbDK3JBmdpOSW1bEyNBdSHPJoeq9Xyts2JN40v
JzU2UxNaLKDaEheWf3F_E52yhHxvMLNdvZJ9FksJdSMK6ZCyGfRJadPN2GhNltqph52sW
iFKUyUk_4RtwXmT_lF49tWOMZqtG-akN9wrBoMsleM0soA0BXIK10rG5cKZoSNr-u2luz
bdZx3CFdAenaqScIkluPPcrXBZGYyX2zYUbGQs2RRXnBmox_yl6CvLbb0qTTYhDnDEo_M
H-ZtWw
```

## Sending SD-JWT and Holder-Selected Disclosures JWT during Presentation

The SD-JWT and the HS-Disclosures JWT can be combined into one document using period character `.` as a separator (here for Example 1):

{#example-simple-combined_sd_jwt_sd_jwt_release}
```
eyJhbGciOiAiUlMyNTYiLCAia2lkIjogImNBRUlVcUowY21MekQxa3pHemhlaUJhZzBZU
kF6VmRsZnhOMjgwTmdIYUEifQ.eyJpc3MiOiAiaHR0cHM6Ly9leGFtcGxlLmNvbS9pc3N
1ZXIiLCAiY25mIjogeyJqd2siOiB7Imt0eSI6ICJSU0EiLCAibiI6ICJwbTRiT0hCZy1v
WWhBeVBXelI1NkFXWDNyVUlYcDExX0lDRGtHZ1M2VzNaV0x0cy1oendJM3g2NTY1OWtnN
GhWbzlkYkdvQ0pFM1pHRl9lYWV0RTMwVWhCVUVncEd3ckRyUWlKOXpxcHJtY0ZmcjNxdn
ZrR2p0dGg4WmdsMWVNMmJKY093RTdQQ0JIV1RLV1lzMTUyUjdnNkpnMk9WcGgtYThycS1
xNzlNaEtHNVFvV19tVHoxMFFUXzZINGM3UGpXRzFmamg4aHBXTm5iUF9wdjZkMXpTd1pm
YzVmbDZ5VlJMMERWMFYzbEdIS2UyV3FmX2VOR2pCckJMVmtsRFRrOC1zdFhfTVdMY1ItR
UdtWEFPdjBVQldpdFNfZFhKS0p1LXZYSnl3MTRuSFNHdXhUSUsyaHgxcHR0TWZ0OUNzdn
FpbVhLZURUVTE0cVFMMWVFN2loY3ciLCAiZSI6ICJBUUFCIn19LCAiaWF0IjogMTUxNjI
zOTAyMiwgImV4cCI6IDE1MTYyNDcwMjIsICJzZF9oYXNoX2FsZyI6ICJzaGEtMjU2Iiwg
InNkX2RpZ2VzdHMiOiB7InN1YiI6ICJPTWR3a2sySFB1aUluUHlwV1VXTXhvdDFZMnRTd
EdzTHVJY0RNaktkWE1VIiwgImdpdmVuX25hbWUiOiAiQWZLS0g0YTBJWmtpOE1GRHl0aE
ZhRlNfWHF6bi13UnZBTWZpeV9WallwRSIsICJmYW1pbHlfbmFtZSI6ICJlVW1YbXJ5MzJ
KaUtfNzZ4TWFzYWdrQVFRc21TVmRXNTdBamsxOHJpU0YwIiwgImVtYWlsIjogIi1SY3I0
ZkR5andsTV9pdGNNeG9RWkNFMVFBRXd5TEpjaWJFcEgxMTRLaUUiLCAicGhvbmVfbnVtY
mVyIjogIkp2Mm53MEMxd1A1QVN1dFlOQXhyV0VuYURSSXBpRjBlVFVBa1VPcDhGNlkiLC
AiYWRkcmVzcyI6ICJacmpLcy1SbUVBVmVBWVN6U3c2R1BGck1wY2djdENmYUo2dDlxUWh
iZko0IiwgImJpcnRoZGF0ZSI6ICJxWFBSUlBkcE5hZWJQOGp0YkVwTy1za0Y0bjd2N0FT
VGg4b0xnMG1rQWRRIn19.QgoJn9wkjFvM9bAr0hTDHLspuqdA21WzfBRVHkASa2ck4PFD
3TC9MiZSi3AiRytRbYT4ZzvkH3BSbm6vy68y62gj0A6OYvZ1Z60Wxho14bxZQveJZgw3u
_lMvYj6GKiUtskypFEHU-Kd-LoDVqEpf6lPQHdpsac__yQ_JL24oCEBlVQRXB-T-6ZNZf
ID6JafSkNNCYQbI8nXbzIEp1LBFm0fE8eUd4G4yPYOj1SeuR6Gy92T0vAoL5QtpIAHo49
oAmiSIj6DQNl2cNYs74jhrBIcNZyt4l8H1lV20wS5OS3T0vXaYD13fgm0p4iWD9cVg3HK
ShUVulEyrSbq94jIKg.eyJhbGciOiAiUlMyNTYiLCAia2lkIjogIkxkeVRYd0F5ZnJpcj
RfVjZORzFSYzEwVThKZExZVHJFQktKaF9oNWlfclUifQ.eyJub25jZSI6ICJYWk9VY28x
dV9nRVBrbnhTNzhzV1dnIiwgImF1ZCI6ICJodHRwczovL2V4YW1wbGUuY29tL3Zlcmlma
WVyIiwgInNkX3JlbGVhc2UiOiB7ImdpdmVuX25hbWUiOiAie1wic1wiOiBcIjZJajd0TS
1hNWlWUEdib1M1dG12VkFcIiwgXCJ2XCI6IFwiSm9oblwifSIsICJmYW1pbHlfbmFtZSI
6ICJ7XCJzXCI6IFwiUWdfTzY0enFBeGU0MTJhMTA4aXJvQVwiLCBcInZcIjogXCJEb2Vc
In0iLCAiYWRkcmVzcyI6ICJ7XCJzXCI6IFwiNWJQczFJcXVaTmEwaGthRnp6elpOd1wiL
CBcInZcIjoge1wic3RyZWV0X2FkZHJlc3NcIjogXCIxMjMgTWFpbiBTdFwiLCBcImxvY2
FsaXR5XCI6IFwiQW55dG93blwiLCBcInJlZ2lvblwiOiBcIkFueXN0YXRlXCIsIFwiY29
1bnRyeVwiOiBcIlVTXCJ9fSJ9fQ.fw4xRl7m1mDPCZvCTn3GOr2PgBZ--fTKfy7s-GuEi
fNvzW5KsJaBBFvzdZztm25XGhk29uw-XwEw00r0hyxXLBvWfA0XbDK3JBmdpOSW1bEyNB
dSHPJoeq9Xyts2JN40vJzU2UxNaLKDaEheWf3F_E52yhHxvMLNdvZJ9FksJdSMK6ZCyGf
RJadPN2GhNltqph52sWiFKUyUk_4RtwXmT_lF49tWOMZqtG-akN9wrBoMsleM0soA0BXI
K10rG5cKZoSNr-u2luzbdZx3CFdAenaqScIkluPPcrXBZGYyX2zYUbGQs2RRXnBmox_yl
6CvLbb0qTTYhDnDEo_MH-ZtWw
```

# Verification and Processing

## Verification by the Holder when Receiving SD-JWT and Issuer-Issued Disclosures Object

The holder SHOULD verify the binding between SD-JWT and II-Disclosures Object by performing the following steps:
 1. Check that all the claims in the II-Disclosures Object are present in the SD-JWT and that there are no claims in the SD-JWT that are not in the II-Disclosures Object
 2. Check that the digests of the claims in the II-Disclosures Object match those in the SD-JWT

## Verification by the Verifier when Receiving SD-JWT and Holder-Selected Disclosures JWT

Verifiers MUST follow [@RFC8725] for validating the SD-JWT and, if signed, the
HS-Disclosures JWT. 

Whether to check the signature of the HS-Disclosures JWT is up to the Verifier's policy,
based on the set of trust requirements such as trust frameworks it belongs to. 
The Verifier MUST NOT accept HS-Disclosures JWTs using "none" algorithm, when the
Verifier's policy requires a signed HS-Disclosures JWT. 

Verifiers MUST go through (at least) the following steps before
trusting/using any of the contents of an SD-JWT:

 1. Determine if holder binding is to be checked for the SD-JWT. Refer to (#holder_binding_security) for details.
 2. Check that the presentation consists of six period-separated (`.`) elements; if holder binding is not required, the last element can be empty.
 3. Separate the SD-JWT from the HS-Disclosures JWT.
 4. Validate the SD-JWT:
    1. Ensure that a signing algorithm was used that was deemed secure for the application. Refer to [@RFC8725], Sections 3.1 and 3.2 for details.
    2. Validate the signature over the SD-JWT.
    3. Validate the issuer of the SD-JWT and that the signing key belongs to this issuer.
    4. Check that the SD-JWT is valid using `nbf`, `iat`, and `exp` claims, if provided in the SD-JWT.
    5. Check that the claim `sd_digests` is present in the SD-JWT.
    6. Check that the `sd_digest_derivation_alg` claim is present and its value is understand
       and the digest derivation algorithm is deemed secure.
 5. Validate the HS-Disclosures JWT:
    1. If holder binding is required, validate the signature over the SD-JWT using the same steps as for the SD-JWT plus the following steps:
      1. Determine that the public key for the private key that used to sign the HS-Disclosures JWT is bound to the SD-JWT, i.e., the SD-JWT either contains a reference to the public key or contains the public key itself.
      2. Determine that the HS-Disclosures JWT is bound to the current transaction and was created for this verifier (replay protection). This is usually achieved by a `nonce` and `aud` field within the HS-Disclosures JWT.
    2. For each claim in the HS-Disclosures JWT:
      1. Ensure that the claim is present as well in `sd_digests` in the SD-JWT.
         If `sd_digests` is structured, the claim MUST be present at the same
         place within the structure.
      2. Compute the base64url-encoded digest of the JSON literal disclosed
         by the Holder using the `sd_digest_derivation_alg` in SD-JWT.
      3. Compare the digests computed in the previous step with the one of
         the same claim in the SD-JWT. Accept the claim only when the two
         digests match.
      4. Ensure that the claim value in the HS-Disclosures JWT is a JSON-encoded
         object containing at least the keys `s` and `v`, and optionally `n`.
      5. Store the value of the key `v` as the claim value. If `n` is contained
         in the object, use the value of the key `n` as the claim name.
    3. Once all necessary claims have been verified, their values can be
       validated and used according to the requirements of the application. It
       MUST be ensured that all claims required for the application have been
       disclosured.

If any step fails, the input is not valid and processing MUST be aborted.

## Processing Model {#processing_model}

Neither an SD-JWT nor an HS-Disclosures JWT is suitable for direct use by an application.
Besides the REQUIRED verification steps listed above, it is further RECOMMENDED
that an application-consumable format is generated from the data released in
the HS-Disclosures. The RECOMMENDED way is to merge the released claims and any
plaintext claims in the SD-JWT recursively:

 * Objects from the released claims must be merged into existing objects from the SD-JWT.
 * If a key is present in both objects:
   * If the value in the released claims is and object and the value in the
     SD-JWT claims is an object, the two objects MUST be merged recursively.
   * Else, the value in the released claims MUST be used.

The keys `sd_digests` and `sd_digest_derivation_alg` SHOULD be removed prior to further
processing.

The processing is shown in Examples 2b and 3 in the Appendix.

# Security Considerations {#security_considerations}

## Mandatory digest computation of the revealed claim values by the Verifier

ToDo: add text explaining mechanisms that should be adopted to ensure that
  verifiers validate the claim values received in HS-Disclosures JWT by calculating the
  digests of those values and comparing them with the digests in the SD-JWT:
  - create a test suite that forces digest computation by the Verifiers,
    and includes negative test cases in test vectors
  - use only implementations/libraries that are compliant to the test suite
  - etc.

## Mandatory signing of the SD-JWT

The SD-JWT MUST be signed by the issuer to protect integrity of the issued
claims. An attacker can modify or add claims if an SD-JWT is not signed (e.g.,
change the "email" attribute to take over the victim's account or add an
attribute indicating a fake academic qualification).

The verifier MUST always check the SD-JWT signature to ensure that the SD-JWT
has not been tampered with since its issuance. If the signature on the SD-JWT
cannot be verified, the SD-JWT MUST be rejected.

## Entropy of the salt

The security model relies on the fact that the salt is not learned or guessed by
the attacker. It is vitally important to adhere to this principle. As such, the
salt MUST be created in such a manner that it is cryptographically random,
long enough and has high entropy that it is not practical for the attacker to
guess. A new salt MUST be chosen for each claim.

## Minimum length of the salt {#salt-minlength}

The RECOMMENDED minimum length of the randomly-generated portion of the salt is 128 bits.

Note that minimum 128 bits would be necessary when SHA-256, HMAC-SHA256, or a function of similar strength is used, but a smaller salt size might achieve similar level of security if a stronger iterative derivation function is used.

The issuer MUST ensure that a new salt value is chosen for each claim,
including when the same claim name occurs at different places in the
structure of the SD-JWT. This can be seen in Example 3 in the Appendix,
where multiple claims with the name `type` appear, but each of them has
a different salt.

## Choice of a digest derivation algorithm

For the security of this scheme, the digest derivation algorithm is required to be preimage and collision
resistant, i.e., it is infeasible to calculate the salt and claim value that result in
a particular digest, and it is infeasible to find a different salt and claim value pair that
result in a matching digest, respectively.

Furthermore the hash algorithms MD2, MD4, MD5, RIPEMD-160, and SHA-1
revealed fundamental weaknesses and they MUST NOT be used.

## Holder Binding {#holder_binding_security}
TBD

## Blinding Claim Names {#blinding-claim-names}

It is RECOMMENDED to use cryptographically random numbers with at least 128 bits
of entropy as placeholder claim names.

With the approach chosen in this specification, claim names of objects
that are not themselves selectively disclosable are not blinded.  This
can be seen in Example 6 in the Appendix, where even in the blinded
SD-JWT, `address` and `delivery_address` are visible. This limitation
needs to be taken into account by issuers when creating the structure of
the SD-JWT.

The issuer MUST ensure that a new random placeholder name is chosen for
each claim, including when the same claim name occurs at different
places in the structure of the SD-JWT. This can be seen in Example 6 in
the Appendix, where multiple claims with same name appear below
`address` and `delivery_address`, but each of them has a different
blinded claim name. For each credential issued, new random placeholder names
MUST be chosen by the issuer.

The order of elements in JSON-encoded objects is not relevant to applications,
but the order may reveal information about the blinded claim name to the
verifier. It is therefore RECOMMENDED to ensure that the order is shuffled or
otherwise hidden (e.g., alphabetically ordered using the blinded claim names).

# Privacy Considerations {#privacy_considerations}

## Claim Names

By default, claim names are not blinded in an SD-JWT. In this case, even when
the claim's value is not known to a verifier, the claim name can disclose some
information to the verifier. For example, if the SD-JWT contains a claim named
`super_secret_club_membership_no`, the verifier might assume that the end-user
is a member of the Super Secret Club.

Blinding claim names can help to avoid this potential privacy issue. In many
cases, however, verifiers can already deduce this or similar information just
from the identification of the issuer and the schema used for the SD-JWT.
Blinding claim names might not provide additional privacy if this is the case.

Furthermore, re-using the same value to blind a claim name may limit the privacy benefits.


## Unlinkability

Colluding issuer/verifier or verifier/verifier pairs could link issuance/presentation or two presentation sessions
to the same user on the basis of unique values encoded in the SD-JWT
(issuer signature, salts, digests, etc.). More advanced cryptographic schemes, outside the scope of
this specification, can be used to prevent this type of linkability.

# Acknowledgements {#Acknowledgements}

We would like to thank
Alen Horvat,
Brian Campbell,
Christian Paquin,
Fabian Hauck,
Giuseppe De Marco,
Kushal Das,
Mike Jones,
Nat Sakimura,
Pieter Kasselman, and
Torsten Lodderstedt
for their contributions (some of which substantial) to this draft and to the initial set of implementations.

The work on this draft was started at OAuth Security Workshop 2022 in Trondheim, Norway.


# IANA Considerations {#iana_considerations}

TBD

<reference anchor="OIDC" target="https://openid.net/specs/openid-connect-core-1_0.html">
  <front>
    <title>OpenID Connect Core 1.0 incorporating errata set 1</title>
    <author initials="N." surname="Sakimura" fullname="Nat Sakimura">
      <organization>NRI</organization>
    </author>
    <author initials="J." surname="Bradley" fullname="John Bradley">
      <organization>Ping Identity</organization>
    </author>
    <author initials="M." surname="Jones" fullname="Mike Jones">
      <organization>Microsoft</organization>
    </author>
    <author initials="B." surname="de Medeiros" fullname="Breno de Medeiros">
      <organization>Google</organization>
    </author>
    <author initials="C." surname="Mortimore" fullname="Chuck Mortimore">
      <organization>Salesforce</organization>
    </author>
   <date day="8" month="Nov" year="2014"/>
  </front>
</reference>

<reference anchor="VC_DATA" target="https://www.w3.org/TR/vc_data">
  <front>
    <title>Verifiable Credentials Data Model 1.0</title>
    <author fullname="Manu Sporny">
      <organization>Digital Bazaar</organization>
    </author>
    <author fullname="Grant Noble">
      <organization>ConsenSys</organization>
    </author>
    <author fullname="Dave Longley">
      <organization>Digital Bazaar</organization>
    </author>
    <author fullname="Daniel C. Burnett">
      <organization>ConsenSys</organization>
    </author>
    <author fullname="Brent Zundel">
      <organization>Evernym</organization>
    </author>
    <author fullname="David Chadwick">
      <organization>University of Kent</organization>
    </author>
    <date day="19" month="Nov" year="2019" />
  </front>
</reference>

{backmatter}

# Additional Examples

All of the following examples are non-normative.

## Example 2a - Structured SD-JWT
This non-normative example is based on the same claim values as Example 1, but
here the issuer decided to create a structured object for the digests. This
allows for the disclosure of individual members of the `address` claim separately.

{#example-simple_structured-sd_jwt_payload}
```json
{
  "iss": "https://example.com/issuer",
  "cnf": {
    "jwk": {
      "kty": "RSA",
      "n": "pm4bOHBg-oYhAyPWzR56AWX3rUIXp11_ICDkGgS6W3ZWLts-hzwI3x656
        59kg4hVo9dbGoCJE3ZGF_eaetE30UhBUEgpGwrDrQiJ9zqprmcFfr3qvvkGjt
        th8Zgl1eM2bJcOwE7PCBHWTKWYs152R7g6Jg2OVph-a8rq-q79MhKG5QoW_mT
        z10QT_6H4c7PjWG1fjh8hpWNnbP_pv6d1zSwZfc5fl6yVRL0DV0V3lGHKe2Wq
        f_eNGjBrBLVklDTk8-stX_MWLcR-EGmXAOv0UBWitS_dXJKJu-vXJyw14nHSG
        uxTIK2hx1pttMft9CsvqimXKeDTU14qQL1eE7ihcw",
      "e": "AQAB"
    }
  },
  "iat": 1516239022,
  "exp": 1516247022,
  "sd_digest_derivation_alg": "sha-256",
  "sd_digests": {
    "sub": "OMdwkk2HPuiInPypWUWMxot1Y2tStGsLuIcDMjKdXMU",
    "given_name": "AfKKH4a0IZki8MFDythFaFS_Xqzn-wRvAMfiy_VjYpE",
    "family_name": "eUmXmry32JiK_76xMasagkAQQsmSVdW57Ajk18riSF0",
    "email": "-Rcr4fDyjwlM_itcMxoQZCE1QAEwyLJcibEpH114KiE",
    "phone_number": "Jv2nw0C1wP5ASutYNAxrWEnaDRIpiF0eTUAkUOp8F6Y",
    "address": {
      "street_address":
        "n25N6kth9N0CwjZXHeth1gfovg8_I8fGyzeY0qeLp0k",
      "locality": "gJVL_TKoT_SbA4_sv0klLTkg-YEGzVUkC-6egxegsz0",
      "region": "zXbstGPuPq2cPJfyD_-HlmqVyFMf03xH-FbeotXxdbo",
      "country": "pN-5CZ5hbumsPvLKUADm4Ott6gu0E4xj09s4Z51yb8U"
    },
    "birthdate": "UxsvgkUgPnawP6wY4hmxJ_jqiNNKni62zrX7hQOUsys"
  }
}
```

The II-Disclosures Object for this SD-JWT is as follows:

{#example-simple_structured-svc_payload}
```json
{
  "sd_ii_disclosures": {
    "sub": "{\"s\": \"2GLC42sKQveCfGfryNRN9w\", \"v\":
      \"6c5c0a49-b589-431d-bae7-219122a9ec2c\"}",
    "given_name": "{\"s\": \"6Ij7tM-a5iVPGboS5tmvVA\", \"v\":
      \"John\"}",
    "family_name": "{\"s\": \"Qg_O64zqAxe412a108iroA\", \"v\":
      \"Doe\"}",
    "email": "{\"s\": \"Pc33JM2LchcU_lHggv_ufQ\", \"v\":
      \"johndoe@example.com\"}",
    "phone_number": "{\"s\": \"lklxF5jMYlGTPUovMNIvCA\", \"v\":
      \"+1-202-555-0101\"}",
    "address": {
      "street_address": "{\"s\": \"5bPs1IquZNa0hkaFzzzZNw\", \"v\":
        \"123 Main St\"}",
      "locality": "{\"s\": \"y1sVU5wdfJahVdgwPgS7RQ\", \"v\":
        \"Anytown\"}",
      "region": "{\"s\": \"C9GSoujviJquEgYfojCb1A\", \"v\":
        \"Anystate\"}",
      "country": "{\"s\": \"H3o1uswP760Fi2yeGdVCEQ\", \"v\": \"US\"}"
    },
    "birthdate": "{\"s\": \"M0Jb57t41ubrkSuyrDT3xA\", \"v\":
      \"1940-01-01\"}"
  }
}
```

An HS-Disclosures JWT for the SD-JWT above that discloses only `region`
and `country` of the `address` property could look as follows:

{#example-simple_structured-sd_jwt_release_payload}
```json
{
  "nonce": "XZOUco1u_gEPknxS78sWWg",
  "aud": "https://example.com/verifier",
  "sd_hs_disclosures": {
    "given_name": "{\"s\": \"6Ij7tM-a5iVPGboS5tmvVA\", \"v\":
      \"John\"}",
    "family_name": "{\"s\": \"Qg_O64zqAxe412a108iroA\", \"v\":
      \"Doe\"}",
    "birthdate": "{\"s\": \"M0Jb57t41ubrkSuyrDT3xA\", \"v\":
      \"1940-01-01\"}",
    "address": {
      "region": "{\"s\": \"C9GSoujviJquEgYfojCb1A\", \"v\":
        \"Anystate\"}",
      "country": "{\"s\": \"H3o1uswP760Fi2yeGdVCEQ\", \"v\": \"US\"}"
    }
  }
}
```

## Example 2b - Mixing SD and Non-SD Claims

In this example, a variant of Example 2a, the issuer decided to apply selective
disclosure only to some of the claims. In particular, the `country` component of
the `address` is contained in the JWT as a regular claim, whereas the rest of
the claims can be disclosed selectively. Note that the processing model
described in (#processing_model) allows for merging the selectively disclosable
claims with the regular claims.

The JSON-payload of the SD-JWT that contains both selectively
disclosable claims in the `sd_digests` object and not-selectively
disclosable claims in a top-level JWT claim would look as follows:

{#example-simple_structured_merging-sd_jwt_payload}
```json
{
  "iss": "https://example.com/issuer",
  "cnf": {
    "jwk": {
      "kty": "RSA",
      "n": "pm4bOHBg-oYhAyPWzR56AWX3rUIXp11_ICDkGgS6W3ZWLts-hzwI3x656
        59kg4hVo9dbGoCJE3ZGF_eaetE30UhBUEgpGwrDrQiJ9zqprmcFfr3qvvkGjt
        th8Zgl1eM2bJcOwE7PCBHWTKWYs152R7g6Jg2OVph-a8rq-q79MhKG5QoW_mT
        z10QT_6H4c7PjWG1fjh8hpWNnbP_pv6d1zSwZfc5fl6yVRL0DV0V3lGHKe2Wq
        f_eNGjBrBLVklDTk8-stX_MWLcR-EGmXAOv0UBWitS_dXJKJu-vXJyw14nHSG
        uxTIK2hx1pttMft9CsvqimXKeDTU14qQL1eE7ihcw",
      "e": "AQAB"
    }
  },
  "iat": 1516239022,
  "exp": 1516247022,
  "sd_digest_derivation_alg": "sha-256",
  "sd_digests": {
    "sub": "OMdwkk2HPuiInPypWUWMxot1Y2tStGsLuIcDMjKdXMU",
    "given_name": "AfKKH4a0IZki8MFDythFaFS_Xqzn-wRvAMfiy_VjYpE",
    "family_name": "eUmXmry32JiK_76xMasagkAQQsmSVdW57Ajk18riSF0",
    "email": "-Rcr4fDyjwlM_itcMxoQZCE1QAEwyLJcibEpH114KiE",
    "phone_number": "Jv2nw0C1wP5ASutYNAxrWEnaDRIpiF0eTUAkUOp8F6Y",
    "address": {
      "street_address":
        "n25N6kth9N0CwjZXHeth1gfovg8_I8fGyzeY0qeLp0k",
      "locality": "gJVL_TKoT_SbA4_sv0klLTkg-YEGzVUkC-6egxegsz0",
      "region": "zXbstGPuPq2cPJfyD_-HlmqVyFMf03xH-FbeotXxdbo"
    },
    "birthdate": "LE_vN7VR3ejbTWb7r_pnvpsNMrvu9i3punsSu0tBEKs"
  },
  "address": {
    "country": "US"
  }
}
```

The holder can now, for example, release the rest of the components of the `address` claim in the HS-Disclosures:


{#example-simple_structured_merging-sd_jwt_release_payload}
```json
{
  "nonce": "XZOUco1u_gEPknxS78sWWg",
  "aud": "https://example.com/verifier",
  "sd_hs_disclosures": {
    "given_name": "{\"s\": \"6Ij7tM-a5iVPGboS5tmvVA\", \"v\":
      \"John\"}",
    "family_name": "{\"s\": \"Qg_O64zqAxe412a108iroA\", \"v\":
      \"Doe\"}",
    "birthdate": "{\"s\": \"M0Jb57t41ubrkSuyrDT3xA\", \"v\":
      \"1940-01-01\"}",
    "address": {
      "region": "{\"s\": \"C9GSoujviJquEgYfojCb1A\", \"v\":
        \"Anystate\"}",
      "street_address": "{\"s\": \"5bPs1IquZNa0hkaFzzzZNw\", \"v\":
        \"123 Main St\"}",
      "locality": "{\"s\": \"y1sVU5wdfJahVdgwPgS7RQ\", \"v\":
        \"Anytown\"}"
    }
  }
}
```

The verifier, after verifying the SD-JWT and applying the HS-Disclosures, would
process the result according to (#processing_model) and pass the following data
to the application:


{#example-simple_structured_merging-merged}
```json
{
  "given_name": "John",
  "family_name": "Doe",
  "birthdate": "1940-01-01",
  "address": {
    "region": "Anystate",
    "street_address": "123 Main St",
    "locality": "Anytown",
    "country": "US"
  },
  "iss": "https://example.com/issuer",
  "cnf": {
    "jwk": {
      "kty": "RSA",
      "n": "pm4bOHBg-oYhAyPWzR56AWX3rUIXp11_ICDkGgS6W3ZWLts-hzwI3x656
        59kg4hVo9dbGoCJE3ZGF_eaetE30UhBUEgpGwrDrQiJ9zqprmcFfr3qvvkGjt
        th8Zgl1eM2bJcOwE7PCBHWTKWYs152R7g6Jg2OVph-a8rq-q79MhKG5QoW_mT
        z10QT_6H4c7PjWG1fjh8hpWNnbP_pv6d1zSwZfc5fl6yVRL0DV0V3lGHKe2Wq
        f_eNGjBrBLVklDTk8-stX_MWLcR-EGmXAOv0UBWitS_dXJKJu-vXJyw14nHSG
        uxTIK2hx1pttMft9CsvqimXKeDTU14qQL1eE7ihcw",
      "e": "AQAB"
    }
  },
  "iat": 1516239022,
  "exp": 1516247022
}
```


## Example 3 - Complex Structured SD-JWT

In this example, a complex object such as those defined in OIDC4IDA
[@OIDC.IDA] is used. Here, the Issuer is using the following user data:

{#example-complex-user_claims}
```json
{
  "verified_claims": {
    "verification": {
      "trust_framework": "de_aml",
      "time": "2012-04-23T18:25Z",
      "verification_process": "f24c6f-6d3f-4ec5-973e-b0d8506f3bc7",
      "evidence": [
        {
          "type": "document",
          "method": "pipp",
          "time": "2012-04-22T11:30Z",
          "document": {
            "type": "idcard",
            "issuer": {
              "name": "Stadt Augsburg",
              "country": "DE"
            },
            "number": "53554554",
            "date_of_issuance": "2010-03-23",
            "date_of_expiry": "2020-03-22"
          }
        }
      ]
    },
    "claims": {
      "given_name": "Max",
      "family_name": "Meier",
      "nationalities": [
        "DE"
      ],
      "address": {
        "locality": "Maxstadt",
        "postal_code": "12344",
        "country": "DE",
        "street_address": "An der Weide 22"
      }
    }
  },
  "birth_middle_name": "Timotheus",
  "salutation": "Dr.",
  "msisdn": "49123456789"
}
```

The issuer in this example further adds the two claims `birthdate` and `place_of_birth` to the `claims` element in plain text. The following shows the resulting SD-JWT payload:

{#example-complex-sd_jwt_payload}
```json
{
  "iss": "https://example.com/issuer",
  "cnf": {
    "jwk": {
      "kty": "RSA",
      "n": "pm4bOHBg-oYhAyPWzR56AWX3rUIXp11_ICDkGgS6W3ZWLts-hzwI3x656
        59kg4hVo9dbGoCJE3ZGF_eaetE30UhBUEgpGwrDrQiJ9zqprmcFfr3qvvkGjt
        th8Zgl1eM2bJcOwE7PCBHWTKWYs152R7g6Jg2OVph-a8rq-q79MhKG5QoW_mT
        z10QT_6H4c7PjWG1fjh8hpWNnbP_pv6d1zSwZfc5fl6yVRL0DV0V3lGHKe2Wq
        f_eNGjBrBLVklDTk8-stX_MWLcR-EGmXAOv0UBWitS_dXJKJu-vXJyw14nHSG
        uxTIK2hx1pttMft9CsvqimXKeDTU14qQL1eE7ihcw",
      "e": "AQAB"
    }
  },
  "iat": 1516239022,
  "exp": 1516247022,
  "sd_digest_derivation_alg": "sha-256",
  "sd_digests": {
    "verified_claims": {
      "verification": {
        "trust_framework":
          "T7ivxsfuy-nAuECeh0utPEX8cSlc7QflJDE0RqtWDMU",
        "time": "_ecCQoXSR8t9esur66ZwWwC6u4xLuVELjmwFgpRZqcQ",
        "verification_process":
          "BolwKKvU8N7uUhjN2aGH2T54wjXpkcOz5sC9PkIP4s4",
        "evidence": [
          {
            "type": "7jBlUZkZn1Gfj9mybqlJGzTb2z8KcNNHU0IV4B8MxOM",
            "method": "BRQgcT09gdBqO-MLTka8d6dlCshZCUNpFgsZoet5I-o",
            "time": "-PVLNSmbkCHLp8S7i077YnHZV0yE8gyKWLpWV2o8FJE",
            "document": {
              "type": "vzDHD-6hQqZ5lSw_7acK1lErxSh3E6dO0zlUYM2hDvw",
              "issuer": {
                "name":
                  "us9T9ufVdSmytSmjrtdN_TUI0ai3_JNM3q-0qx0CXk4",
                "country":
                  "uItKtPRZQBB9v5THHOdi02ALjD0MH0U6jjHDLe91NnY"
              },
              "number":
                "QNNXwo3siOWdqNivKBnFsD4X8gZxVIgu3tv6dfpZhUc",
              "date_of_issuance":
                "AYWQphnOlFFN9oSVvtBr_iYCKYlucTi3lsMrXebebgc",
              "date_of_expiry":
                "JIk-APYHW3qy60rvGyFswDCTMfAbBXZyyrZEn8NsBhU"
            }
          }
        ]
      },
      "claims": {
        "given_name": "hZtT6FZBzxAeByDUkFJTeqTCpTd2cQKx6MDPkGvVCRE",
        "family_name": "5yLYGVxPSfXynhcopbIcrFe0_sMGxv_-6THZAu4eWnU",
        "nationalities":
          "BxCtneHl-RQoL24tS8AaywfyHpnZSq9tUsNDyrYFLYY",
        "address": {
          "locality": "ah6QI8ceduHKP7uiHbwZ2a2LYkxjibHaoWG3M6x1ip4",
          "postal_code":
            "Auci5Y0jrp_3ahg_IW_Z-mqBaE9BrrItR6o7ekhEGBo",
          "country": "RAKTJg_m1tcoyGI1O2qgQm4KD2d2abXhU4IS7c6RVjU",
          "street_address":
            "iKkk1nJHTBKTkEt2TNMkZf69WYkiDYaQL6ZzDZmGO1M"
        }
      }
    },
    "birth_middle_name":
      "KpRjGCm3uykvCGFIDrVJ7iTMQhWakBmCItHbAa6vnZE",
    "salutation": "IoY5e03e65CUrnaMcRDmPCm0RWPEFE4mVkoCsK86agA",
    "msisdn": "XupJick4P8bxaz20kx_VOwbGU1cgslhAUG6IE-tDjms"
  },
  "verified_claims": {
    "claims": {
      "birthdate": "1956-01-28",
      "place_of_birth": {
        "country": "DE",
        "locality": "Musterstadt"
      }
    }
  }
}
```

The SD-JWT is then signed by the issuer to create a document like the following:

{#example-complex-serialized_sd_jwt}
```
eyJhbGciOiAiUlMyNTYiLCAia2lkIjogImNBRUlVcUowY21MekQxa3pHemhlaUJhZzBZU
kF6VmRsZnhOMjgwTmdIYUEifQ.eyJpc3MiOiAiaHR0cHM6Ly9leGFtcGxlLmNvbS9pc3N
1ZXIiLCAiY25mIjogeyJqd2siOiB7Imt0eSI6ICJSU0EiLCAibiI6ICJwbTRiT0hCZy1v
WWhBeVBXelI1NkFXWDNyVUlYcDExX0lDRGtHZ1M2VzNaV0x0cy1oendJM3g2NTY1OWtnN
GhWbzlkYkdvQ0pFM1pHRl9lYWV0RTMwVWhCVUVncEd3ckRyUWlKOXpxcHJtY0ZmcjNxdn
ZrR2p0dGg4WmdsMWVNMmJKY093RTdQQ0JIV1RLV1lzMTUyUjdnNkpnMk9WcGgtYThycS1
xNzlNaEtHNVFvV19tVHoxMFFUXzZINGM3UGpXRzFmamg4aHBXTm5iUF9wdjZkMXpTd1pm
YzVmbDZ5VlJMMERWMFYzbEdIS2UyV3FmX2VOR2pCckJMVmtsRFRrOC1zdFhfTVdMY1ItR
UdtWEFPdjBVQldpdFNfZFhKS0p1LXZYSnl3MTRuSFNHdXhUSUsyaHgxcHR0TWZ0OUNzdn
FpbVhLZURUVTE0cVFMMWVFN2loY3ciLCAiZSI6ICJBUUFCIn19LCAiaWF0IjogMTUxNjI
zOTAyMiwgImV4cCI6IDE1MTYyNDcwMjIsICJzZF9kaWdlc3RfZGVyaXZhdGlvbl9hbGci
OiAic2hhLTI1NiIsICJzZF9kaWdlc3RzIjogeyJ2ZXJpZmllZF9jbGFpbXMiOiB7InZlc
mlmaWNhdGlvbiI6IHsidHJ1c3RfZnJhbWV3b3JrIjogIlQ3aXZ4c2Z1eS1uQXVFQ2VoMH
V0UEVYOGNTbGM3UWZsSkRFMFJxdFdETVUiLCAidGltZSI6ICJfZWNDUW9YU1I4dDllc3V
yNjZad1d3QzZ1NHhMdVZFTGptd0ZncFJacWNRIiwgInZlcmlmaWNhdGlvbl9wcm9jZXNz
IjogIkJvbHdLS3ZVOE43dVVoak4yYUdIMlQ1NHdqWHBrY096NXNDOVBrSVA0czQiLCAiZ
XZpZGVuY2UiOiBbeyJ0eXBlIjogIjdqQmxVWmtabjFHZmo5bXlicWxKR3pUYjJ6OEtjTk
5IVTBJVjRCOE14T00iLCAibWV0aG9kIjogIkJSUWdjVDA5Z2RCcU8tTUxUa2E4ZDZkbEN
zaFpDVU5wRmdzWm9ldDVJLW8iLCAidGltZSI6ICItUFZMTlNtYmtDSExwOFM3aTA3N1lu
SFpWMHlFOGd5S1dMcFdWMm84RkpFIiwgImRvY3VtZW50IjogeyJ0eXBlIjogInZ6REhEL
TZoUXFaNWxTd183YWNLMWxFcnhTaDNFNmRPMHpsVVlNMmhEdnciLCAiaXNzdWVyIjogey
JuYW1lIjogInVzOVQ5dWZWZFNteXRTbWpydGROX1RVSTBhaTNfSk5NM3EtMHF4MENYazQ
iLCAiY291bnRyeSI6ICJ1SXRLdFBSWlFCQjl2NVRISE9kaTAyQUxqRDBNSDBVNmpqSERM
ZTkxTm5ZIn0sICJudW1iZXIiOiAiUU5OWHdvM3NpT1dkcU5pdktCbkZzRDRYOGdaeFZJZ
3UzdHY2ZGZwWmhVYyIsICJkYXRlX29mX2lzc3VhbmNlIjogIkFZV1FwaG5PbEZGTjlvU1
Z2dEJyX2lZQ0tZbHVjVGkzbHNNclhlYmViZ2MiLCAiZGF0ZV9vZl9leHBpcnkiOiAiSkl
rLUFQWUhXM3F5NjBydkd5RnN3RENUTWZBYkJYWnl5clpFbjhOc0JoVSJ9fV19LCAiY2xh
aW1zIjogeyJnaXZlbl9uYW1lIjogImhadFQ2RlpCenhBZUJ5RFVrRkpUZXFUQ3BUZDJjU
Ut4Nk1EUGtHdlZDUkUiLCAiZmFtaWx5X25hbWUiOiAiNXlMWUdWeFBTZlh5bmhjb3BiSW
NyRmUwX3NNR3h2Xy02VEhaQXU0ZVduVSIsICJuYXRpb25hbGl0aWVzIjogIkJ4Q3RuZUh
sLVJRb0wyNHRTOEFheXdmeUhwblpTcTl0VXNORHlyWUZMWVkiLCAiYWRkcmVzcyI6IHsi
bG9jYWxpdHkiOiAiYWg2UUk4Y2VkdUhLUDd1aUhid1oyYTJMWWt4amliSGFvV0czTTZ4M
WlwNCIsICJwb3N0YWxfY29kZSI6ICJBdWNpNVkwanJwXzNhaGdfSVdfWi1tcUJhRTlCcn
JJdFI2bzdla2hFR0JvIiwgImNvdW50cnkiOiAiUkFLVEpnX20xdGNveUdJMU8ycWdRbTR
LRDJkMmFiWGhVNElTN2M2UlZqVSIsICJzdHJlZXRfYWRkcmVzcyI6ICJpS2trMW5KSFRC
S1RrRXQyVE5Na1pmNjlXWWtpRFlhUUw2WnpEWm1HTzFNIn19fSwgImJpcnRoX21pZGRsZ
V9uYW1lIjogIktwUmpHQ20zdXlrdkNHRklEclZKN2lUTVFoV2FrQm1DSXRIYkFhNnZuWk
UiLCAic2FsdXRhdGlvbiI6ICJJb1k1ZTAzZTY1Q1VybmFNY1JEbVBDbTBSV1BFRkU0bVZ
rb0NzSzg2YWdBIiwgIm1zaXNkbiI6ICJYdXBKaWNrNFA4YnhhejIwa3hfVk93YkdVMWNn
c2xoQVVHNklFLXREam1zIn0sICJ2ZXJpZmllZF9jbGFpbXMiOiB7ImNsYWltcyI6IHsiY
mlydGhkYXRlIjogIjE5NTYtMDEtMjgiLCAicGxhY2Vfb2ZfYmlydGgiOiB7ImNvdW50cn
kiOiAiREUiLCAibG9jYWxpdHkiOiAiTXVzdGVyc3RhZHQifX19fQ.jB5q1Ogo1so3jKPi
1xeYgBtqrH7VLChOilTp9kw500Vp0sEZzMeLu4Qd7ZDYf6_DqzDuTusHUAlY8pfS56vXX
EZfv9vbWBucdiBp2FUr7izo5TSRpndBc9OH8CKvML6OouZYrDwrCmMcdJcPlf5Zvzr82l
c4q_Rzz-ER49UiAU0RP0BOMutMvM58lHVBzj_NbnXUFMaLZcYGp9Gp7KVnygojkFgJxrC
JMZh_uwDaCUuu81jBnsDeewtN7yhA-IEWZJO6BqLVmVkk1knYzf1lXEwrEzsgeRF2F8_q
ZEpnwkYKcDIxj43Hev1a05e-vZYBqSU44GyjYEmUMo8kN-_eEw
```

An HS-Disclosures JWT for some of the claims may look as follows:

{#example-complex-sd_jwt_release_payload}
```json
{
  "nonce": "XZOUco1u_gEPknxS78sWWg",
  "aud": "https://example.com/verifier",
  "sd_hs_disclosures": {
    "verified_claims": {
      "verification": {
        "trust_framework": "{\"s\": \"2GLC42sKQveCfGfryNRN9w\",
          \"v\": \"de_aml\"}",
        "time": "{\"s\": \"6Ij7tM-a5iVPGboS5tmvVA\", \"v\":
          \"2012-04-23T18:25Z\"}",
        "evidence": [
          {
            "type": "{\"s\": \"Pc33JM2LchcU_lHggv_ufQ\", \"v\":
              \"document\"}"
          }
        ]
      },
      "claims": {
        "given_name": "{\"s\": \"4KyR32oIZt-zkWvFqbULKg\", \"v\":
          \"Max\"}",
        "family_name": "{\"s\": \"flNP1ncMz9Lg-c9qMIz_9g\", \"v\":
          \"Meier\"}"
      }
    }
  }
}
```

After verifying the SD-JWT and HS-Disclosures, the verifier merges the selectively
disclosed claims into the other data contained in the JWT. The verifier will
then pass the result on to the application for further processing:

{#example-complex-merged}
```json
{
  "verified_claims": {
    "verification": {
      "trust_framework": "de_aml",
      "time": "2012-04-23T18:25Z",
      "evidence": [
        {
          "type": "document"
        }
      ]
    },
    "claims": {
      "given_name": "Max",
      "family_name": "Meier",
      "birthdate": "1956-01-28",
      "place_of_birth": {
        "country": "DE",
        "locality": "Musterstadt"
      }
    }
  },
  "iss": "https://example.com/issuer",
  "cnf": {
    "jwk": {
      "kty": "RSA",
      "n": "pm4bOHBg-oYhAyPWzR56AWX3rUIXp11_ICDkGgS6W3ZWLts-hzwI3x656
        59kg4hVo9dbGoCJE3ZGF_eaetE30UhBUEgpGwrDrQiJ9zqprmcFfr3qvvkGjt
        th8Zgl1eM2bJcOwE7PCBHWTKWYs152R7g6Jg2OVph-a8rq-q79MhKG5QoW_mT
        z10QT_6H4c7PjWG1fjh8hpWNnbP_pv6d1zSwZfc5fl6yVRL0DV0V3lGHKe2Wq
        f_eNGjBrBLVklDTk8-stX_MWLcR-EGmXAOv0UBWitS_dXJKJu-vXJyw14nHSG
        uxTIK2hx1pttMft9CsvqimXKeDTU14qQL1eE7ihcw",
      "e": "AQAB"
    }
  },
  "iat": 1516239022,
  "exp": 1516247022
}
```

## Example 4 - W3C Verifiable Credentials Data Model

This example illustrates how the artifacts defined in this specification can be
represented using W3C Verifiable Credentials Data Model as defined in
[@VC_DATA].

SD-JWT is equivalent to an issuer-signed W3C Verifiable Credential (VC). II-Disclosures Object is sent alongside a VC.

HS-Disclosures JWT is equivalent to a holder-signed W3C Verifiable Presentation (VP).

HS-Disclosures JWT as a VP contains a `verifiableCredential` claim inside a `vp` claim that is a string array of an SD-JWT as a VC using JWT compact serialization.

Below is a non-normative example of an SD-JWT represented as a verifiable credential
encoded as JSON and signed as JWS compliant to [@VC_DATA].

II-Disclosures Object sent alongside this SD-JWT as a JWT-VC is same as in Example 1.

```json
{
  "sub": "urn:ietf:params:oauth:jwk-thumbprint:sha-256:NzbLsXh8uDCc
    d-6MNwXF4W_7noWXFZAfHkxZsRGC9Xs",
  "jti": "http://example.edu/credentials/3732",
  "iss": "https://example.com/keys/foo.jwk",
  "nbf": 1541493724,
  "iat": 1541493724,
  "exp": 1573029723,
  "cnf": {
    "jwk": {
      "kty":"RSA",
      "n": "0vx7agoebGcQSuuPiLJXZptN9nndrQmbXEps2aiAFbWhM78LhWx
     4cbbfAAtVT86zwu1RK7aPFFxuhDR1L6tSoc_BJECPebWKRXjBZCiFV4n3oknjhMs
     tn64tZ_2W-5JsGY4Hc5n9yBXArwl93lqt7_RN5w6Cf0h4QyQ5v-65YGjQR0_FDW2
     QvzqY368QQMicAtaSqzs8KJZgnYb9c7d0zgdAZHzu6qMQvRL5hajrn1n91CbOpbI
     SD08qNLyrdkt-bFTWhAI4vMQFh6WeZu0fM4lFd2NcRwr3XPksINHaQ-G_xBniIqb
     w0Ls1jF44-csFCur-kEgU8awapJzKnqDKgw",
      "e":"AQAB"
    }
  },
  "vc": {
    "@context": [
      "https://www.w3.org/2018/credentials/v1",
      "https://www.w3.org/2018/credentials/examples/v1"
    ],
    "type": [
      "VerifiableCredential",
      "UniversityDegreeCredential"
    ],
    "credentialSubject": {
      "first_name": "Jane",
      "last_name": "Doe"
    }
  },
  "sd_digests": {
    "vc": {
      "credentialSubject": {
        "email": "ET2A1JQLF85ZpBulh6UFstGrSfR4B3KM-bjQVllhxqY",
        "phone_number": "SJnciB2DIRVA5cXBrdKoH6n45788mZyUn2rnv74
          uMVU",
        "address": "0FldqLfGnERPPVDC17od9xb4w3iRJTEQbW_Yk9AmnDw",
        "birthdate": "-L0kMgIbLXe3OEkKTUGwz_QKhjehDeofKGwoPrxLuo4"
      }
    }
  }
}
```

Below is a non-normative example of a HS-Disclosures JWT represented as a verifiable presentation
encoded as JSON and signed as a JWS compliant to [@VC_DATA].

```json
{
  "alg": "RS256",
  "typ": "JWT",
  "jwk": {
      "kty":"RSA",
      "n": "0vx7agoebGcQSuuPiLJXZptN9nndrQmbXEps2aiAFbWhM78LhWx
     4cbbfAAtVT86zwu1RK7aPFFxuhDR1L6tSoc_BJECPebWKRXjBZCiFV4n3oknjhMs
     tn64tZ_2W-5JsGY4Hc5n9yBXArwl93lqt7_RN5w6Cf0h4QyQ5v-65YGjQR0_FDW2
     QvzqY368QQMicAtaSqzs8KJZgnYb9c7d0zgdAZHzu6qMQvRL5hajrn1n91CbOpbI
     SD08qNLyrdkt-bFTWhAI4vMQFh6WeZu0fM4lFd2NcRwr3XPksINHaQ-G_xBniIqb
     w0Ls1jF44-csFCur-kEgU8awapJzKnqDKgw",
      "e":"AQAB"
    }
}.{
  "iss": "urn:ietf:params:oauth:jwk-thumbprint:sha-256:NzbLsXh8uDCc
    d-6MNwXF4W_7noWXFZAfHkxZsRGC9Xs",
  "aud": "s6BhdRkqt3",
  "nbf": 1560415047,
  "iat": 1560415047,
  "exp": 1573029723,
  "nonce": "660!6345FSer",
  "vp": {
    "@context": [
      "https://www.w3.org/2018/credentials/v1"
    ],
    "type": [
      "VerifiablePresentation"
    ],
    "verifiableCredential": ["eyJhb...npyXw"]
  },
  "sd_hs_disclosures": {
    "email": "[\"eI8ZWm9QnKPpNPeNenHdhQ\", \"johndoe@example.com\"]",
    "phone_number": "[\"Qg_O64zqAxe412a108iroA\",
      \"+1-202-555-0101\"]",
    "address": "[\"AJx-095VPrpTtN4QMOqROA\", {\"street_address\":
      \"123 Main St\", \"locality\": \"Anytown\", \"region\":
      \"Anystate\", \"country\": \"US\"}]",
    "birthdate": "[\"Pc33JM2LchcU_lHggv_ufQ\", \"1940-01-01\"]"
  }
}
```
## Blinding Claim Names

The following examples show the use of blinded claim names.

### Example 5: Some Blinded Claims

The following shows the user information used in this example, included a claim named `secret_club_membership_no`:

{#example-simple_structured_some_blinded-user_claims}
```json
{
  "sub": "6c5c0a49-b589-431d-bae7-219122a9ec2c",
  "given_name": "John",
  "family_name": "Doe",
  "email": "johndoe@example.com",
  "phone_number": "+1-202-555-0101",
  "secret_club_membership_no": "23",
  "other_secret_club_membership_no": "42",
  "address": {
    "street_address": "123 Main St",
    "locality": "Anytown",
    "region": "Anystate",
    "country": "US"
  },
  "birthdate": "1940-01-01"
}
```

Hiding just the claim `secret_club_membership_no`, the following SD-JWT payload would result:

{#example-simple_structured_some_blinded-sd_jwt_payload}
```json
{
  "iss": "https://example.com/issuer",
  "cnf": {
    "jwk": {
      "kty": "RSA",
      "n": "pm4bOHBg-oYhAyPWzR56AWX3rUIXp11_ICDkGgS6W3ZWLts-hzwI3x656
        59kg4hVo9dbGoCJE3ZGF_eaetE30UhBUEgpGwrDrQiJ9zqprmcFfr3qvvkGjt
        th8Zgl1eM2bJcOwE7PCBHWTKWYs152R7g6Jg2OVph-a8rq-q79MhKG5QoW_mT
        z10QT_6H4c7PjWG1fjh8hpWNnbP_pv6d1zSwZfc5fl6yVRL0DV0V3lGHKe2Wq
        f_eNGjBrBLVklDTk8-stX_MWLcR-EGmXAOv0UBWitS_dXJKJu-vXJyw14nHSG
        uxTIK2hx1pttMft9CsvqimXKeDTU14qQL1eE7ihcw",
      "e": "AQAB"
    }
  },
  "iat": 1516239022,
  "exp": 1516247022,
  "sd_digest_derivation_alg": "sha-256",
  "sd_digests": {
    "sub": "OMdwkk2HPuiInPypWUWMxot1Y2tStGsLuIcDMjKdXMU",
    "given_name": "AfKKH4a0IZki8MFDythFaFS_Xqzn-wRvAMfiy_VjYpE",
    "family_name": "eUmXmry32JiK_76xMasagkAQQsmSVdW57Ajk18riSF0",
    "email": "-Rcr4fDyjwlM_itcMxoQZCE1QAEwyLJcibEpH114KiE",
    "phone_number": "Jv2nw0C1wP5ASutYNAxrWEnaDRIpiF0eTUAkUOp8F6Y",
    "5a2W0_NrlEZzfqmk_7Pq-w":
      "gc8VzGTImYRXzP6j7q5RomXt2C_wtsOJ3hAHJdTuEIY",
    "other_secret_club_membership_no":
      "IirAwgN-MubteYvJ4fmq04p9PnpRTf7hqg0dzSWRboA",
    "address": {
      "street_address":
        "o_yJIdfhKuKVzOF7i1EuakzC5ghd99CX8_nitm-DsRM",
      "locality": "ogNqsvRqK0-ZPZc9C3Z4_6APvywm-lrm0oF2gcVtl_4",
      "region": "8kFihRLSkEheK0zbEsQ3zKXt8csE6OXJE_jv3032BbU",
      "country": "11IMcoA18LrFSpbysx-uqe7N3I3-QZKwCJqYeQuOUY4"
    },
    "birthdate": "PNtcyxm0Q5PyiBuG4f6eAbK6h4tF2FffwG3xqknZ_5A"
  }
}
```

In the II-Disclosures Object, it can be seen that the blinded claim's original name is `secret_club_membership_no`:


{#example-simple_structured_some_blinded-svc_payload}
```json
{
  "sd_ii_disclosures": {
    "sub": "{\"s\": \"2GLC42sKQveCfGfryNRN9w\", \"v\":
      \"6c5c0a49-b589-431d-bae7-219122a9ec2c\"}",
    "given_name": "{\"s\": \"6Ij7tM-a5iVPGboS5tmvVA\", \"v\":
      \"John\"}",
    "family_name": "{\"s\": \"Qg_O64zqAxe412a108iroA\", \"v\":
      \"Doe\"}",
    "email": "{\"s\": \"Pc33JM2LchcU_lHggv_ufQ\", \"v\":
      \"johndoe@example.com\"}",
    "phone_number": "{\"s\": \"lklxF5jMYlGTPUovMNIvCA\", \"v\":
      \"+1-202-555-0101\"}",
    "5a2W0_NrlEZzfqmk_7Pq-w": "{\"s\": \"5bPs1IquZNa0hkaFzzzZNw\",
      \"v\": \"23\", \"n\": \"secret_club_membership_no\"}",
    "other_secret_club_membership_no": "{\"s\":
      \"y1sVU5wdfJahVdgwPgS7RQ\", \"v\": \"42\"}",
    "address": {
      "street_address": "{\"s\": \"C9GSoujviJquEgYfojCb1A\", \"v\":
        \"123 Main St\"}",
      "locality": "{\"s\": \"H3o1uswP760Fi2yeGdVCEQ\", \"v\":
        \"Anytown\"}",
      "region": "{\"s\": \"M0Jb57t41ubrkSuyrDT3xA\", \"v\":
        \"Anystate\"}",
      "country": "{\"s\": \"eK5o5pHfgupPpltj1qhAJw\", \"v\": \"US\"}"
    },
    "birthdate": "{\"s\": \"WpxJrFuX8uSi2p4ht09jvw\", \"v\":
      \"1940-01-01\"}"
  }
}
```

The verifier would learn this information via the HS-Disclosures JWT:

{#example-simple_structured_some_blinded-sd_jwt_release_payload}
```json
{
  "nonce": "XZOUco1u_gEPknxS78sWWg",
  "aud": "https://example.com/verifier",
  "sd_hs_disclosures": {
    "given_name": "{\"s\": \"6Ij7tM-a5iVPGboS5tmvVA\", \"v\":
      \"John\"}",
    "family_name": "{\"s\": \"Qg_O64zqAxe412a108iroA\", \"v\":
      \"Doe\"}",
    "birthdate": "{\"s\": \"WpxJrFuX8uSi2p4ht09jvw\", \"v\":
      \"1940-01-01\"}",
    "address": {
      "region": "{\"s\": \"M0Jb57t41ubrkSuyrDT3xA\", \"v\":
        \"Anystate\"}",
      "country": "{\"s\": \"eK5o5pHfgupPpltj1qhAJw\", \"v\": \"US\"}"
    },
    "5a2W0_NrlEZzfqmk_7Pq-w": "{\"s\": \"5bPs1IquZNa0hkaFzzzZNw\",
      \"v\": \"23\", \"n\": \"secret_club_membership_no\"}"
  }
}
```

The verifier would decode the data as follows:


{#example-simple_structured_some_blinded-verified_contents}
```json
{
  "given_name": "John",
  "family_name": "Doe",
  "birthdate": "1940-01-01",
  "address": {
    "region": "Anystate",
    "country": "US"
  },
  "secret_club_membership_no": "23"
}
```
### Example 6: All Claim Names Blinded

In this example, all claim names are blinded. The user data includes a
non-standard `delivery_address` claim to show that even though the same
claim name appears at different places within the structure, different
salts and blinded claim names are used for them:

{#example-simple_structured_all_blinded-user_claims}
```json
{
  "sub": "6c5c0a49-b589-431d-bae7-219122a9ec2c",
  "given_name": "John",
  "family_name": "Doe",
  "email": "johndoe@example.com",
  "phone_number": "+1-202-555-0101",
  "secret_club_membership_no": "23",
  "address": {
    "street_address": "123 Main St",
    "locality": "Anytown",
    "region": "Anystate",
    "country": "US"
  },
  "delivery_address": {
    "street_address": "123 Main St",
    "locality": "Anytown",
    "region": "Anystate",
    "country": "US"
  },
  "birthdate": "1940-01-01"
}
```


The resulting SD-JWT payload:

{#example-simple_structured_all_blinded-sd_jwt_payload}
```json
{
  "iss": "https://example.com/issuer",
  "cnf": {
    "jwk": {
      "kty": "RSA",
      "n": "pm4bOHBg-oYhAyPWzR56AWX3rUIXp11_ICDkGgS6W3ZWLts-hzwI3x656
        59kg4hVo9dbGoCJE3ZGF_eaetE30UhBUEgpGwrDrQiJ9zqprmcFfr3qvvkGjt
        th8Zgl1eM2bJcOwE7PCBHWTKWYs152R7g6Jg2OVph-a8rq-q79MhKG5QoW_mT
        z10QT_6H4c7PjWG1fjh8hpWNnbP_pv6d1zSwZfc5fl6yVRL0DV0V3lGHKe2Wq
        f_eNGjBrBLVklDTk8-stX_MWLcR-EGmXAOv0UBWitS_dXJKJu-vXJyw14nHSG
        uxTIK2hx1pttMft9CsvqimXKeDTU14qQL1eE7ihcw",
      "e": "AQAB"
    }
  },
  "iat": 1516239022,
  "exp": 1516247022,
  "sd_digest_derivation_alg": "sha-256",
  "sd_digests": {
    "eluV5Og3gSNII8EYnsxA_A":
      "bvPLqohL5ROmk2UsuNffH8C1wx9o-ipm-G4SkUwrpAE",
    "eI8ZWm9QnKPpNPeNenHdhQ":
      "pCtjs0hC2Klhsnpe7BIqnGAsXlyXXC-lAEgX6isoYVM",
    "AJx-095VPrpTtN4QMOqROA":
      "HS1Ht-bTrXsSTw9JdcHIbTFDkEI_IY52_cmzUgxWZ0k",
    "G02NSrQfjFXQ7Io09syajA":
      "M2YQ_j8OPPBK3ZLhPPP6_AdSa2-rug2urYjgk_ML_QM",
    "nPuoQnkRFq3BIeAm7AnXFA":
      "-Brzrp2cs-8nLs7rQI89YJ76s3PrbVe3n_5hlYCy1cE",
    "5a2W0_NrlEZzfqmk_7Pq-w":
      "gc8VzGTImYRXzP6j7q5RomXt2C_wtsOJ3hAHJdTuEIY",
    "address": {
      "HbQ4X8srVW3QDxnIJdqyOA":
        "39o5dKobVi8c0dLpg4sjd7zW18UONRra0ht9mgu4hec",
      "kx5kF17V-x0JmwUx9vgvtw":
        "wqueD5ABJ3bTyGSckOMpzI7YUvcCO2l-40vi6JMYsYY",
      "OBKlTVlvLg-AdwqYGbP8ZA":
        "S11dsdFN97YtrA2o3yZ0eBbf1zn-izejORU-fyMtynI",
      "DsmtKNgpV4dAHpjrcaosAw":
        "-0XEQHSNzMu244QaOpLmPD3JkdZN8SrqbEQ4VDufu9A"
    },
    "delivery_address": {
      "j7ADdb0UVb0Li0ciPcP0ew":
        "mZNJT4TGOf8CxvON6boNM4q5JmOToJyg93yfnP7-fpc",
      "atSmFACYMbJVKD05o3JgtQ":
        "XToTx7QsB7rtaO0OO60hJ9ZolyzcmuXrr8wwBGJKQR8",
      "chBCsyhyh-J86I-awQDiCQ":
        "XZiMd58TainQwxm2CUWtH0nhnNs1CYgvEzCKsRGuQE4",
      "ovoV1rkh_ANm861qUAA2Aw":
        "2O1cfKdHLv75PYQySKbezwOtSbYthspwz_WIbRNjhFk"
    },
    "R7D6Lhl52uxloBQFRulzzA":
      "vMUNkAiWMOKbyS_yDXOy2pU2vXWgd-FfVKq-wVztVc0"
  }
}
```

The II-Disclosures Object:
{#example-simple_structured_all_blinded-svc_payload}
```json
{
  "sd_ii_disclosures": {
    "eluV5Og3gSNII8EYnsxA_A": "{\"s\": \"2GLC42sKQveCfGfryNRN9w\",
      \"v\": \"6c5c0a49-b589-431d-bae7-219122a9ec2c\", \"n\":
      \"sub\"}",
    "eI8ZWm9QnKPpNPeNenHdhQ": "{\"s\": \"6Ij7tM-a5iVPGboS5tmvVA\",
      \"v\": \"John\", \"n\": \"given_name\"}",
    "AJx-095VPrpTtN4QMOqROA": "{\"s\": \"Qg_O64zqAxe412a108iroA\",
      \"v\": \"Doe\", \"n\": \"family_name\"}",
    "G02NSrQfjFXQ7Io09syajA": "{\"s\": \"Pc33JM2LchcU_lHggv_ufQ\",
      \"v\": \"johndoe@example.com\", \"n\": \"email\"}",
    "nPuoQnkRFq3BIeAm7AnXFA": "{\"s\": \"lklxF5jMYlGTPUovMNIvCA\",
      \"v\": \"+1-202-555-0101\", \"n\": \"phone_number\"}",
    "5a2W0_NrlEZzfqmk_7Pq-w": "{\"s\": \"5bPs1IquZNa0hkaFzzzZNw\",
      \"v\": \"23\", \"n\": \"secret_club_membership_no\"}",
    "address": {
      "HbQ4X8srVW3QDxnIJdqyOA": "{\"s\": \"y1sVU5wdfJahVdgwPgS7RQ\",
        \"v\": \"123 Main St\", \"n\": \"street_address\"}",
      "kx5kF17V-x0JmwUx9vgvtw": "{\"s\": \"C9GSoujviJquEgYfojCb1A\",
        \"v\": \"Anytown\", \"n\": \"locality\"}",
      "OBKlTVlvLg-AdwqYGbP8ZA": "{\"s\": \"H3o1uswP760Fi2yeGdVCEQ\",
        \"v\": \"Anystate\", \"n\": \"region\"}",
      "DsmtKNgpV4dAHpjrcaosAw": "{\"s\": \"M0Jb57t41ubrkSuyrDT3xA\",
        \"v\": \"US\", \"n\": \"country\"}"
    },
    "j7ADdb0UVb0Li0ciPcP0ew": "{\"s\": \"eK5o5pHfgupPpltj1qhAJw\",
      \"v\": \"1940-01-01\", \"n\": \"birthdate\"}"
  }
}
```

Here, the holder decided only to disclose a subset of the claims to the verifier:

{#example-simple_structured_all_blinded-sd_jwt_release_payload}
```json
{
  "nonce": "XZOUco1u_gEPknxS78sWWg",
  "aud": "https://example.com/verifier",
  "sd_hs_disclosures": {
    "eI8ZWm9QnKPpNPeNenHdhQ": "{\"s\": \"6Ij7tM-a5iVPGboS5tmvVA\",
      \"v\": \"John\", \"n\": \"given_name\"}",
    "AJx-095VPrpTtN4QMOqROA": "{\"s\": \"Qg_O64zqAxe412a108iroA\",
      \"v\": \"Doe\", \"n\": \"family_name\"}",
    "j7ADdb0UVb0Li0ciPcP0ew": "{\"s\": \"eK5o5pHfgupPpltj1qhAJw\",
      \"v\": \"1940-01-01\", \"n\": \"birthdate\"}",
    "address": {
      "OBKlTVlvLg-AdwqYGbP8ZA": "{\"s\": \"H3o1uswP760Fi2yeGdVCEQ\",
        \"v\": \"Anystate\", \"n\": \"region\"}",
      "DsmtKNgpV4dAHpjrcaosAw": "{\"s\": \"M0Jb57t41ubrkSuyrDT3xA\",
        \"v\": \"US\", \"n\": \"country\"}"
    }
  }
}
```

The verifier would decode the HS-Disclosures JWT and SD-JWT as follows:


{#example-simple_structured_all_blinded-verified_contents}
```json
{
  "given_name": "John",
  "family_name": "Doe",
  "birthdate": "1940-01-01",
  "address": {
    "region": "Anystate",
    "country": "US"
  }
}
```




# Document History

   [[ To be removed from the final specification ]]

   -01

   * introduce blinded claim names
   * explain why JSON-encoding of values is needed
   * explain merging algorithm ("processing model")
   * generalized hash alg to digest derivation alg which also enables HMAC to calculate digests
   * `sd_digest_derivation_alg` renamed to `sd_digest_derivation_alg`
   * Salt/Value Container (SVC) renamed to Issuer-Issued Disclosures (II-Disclosures)
   * SD-JWT-Release (SD-JWT-R) renamed to Holder-Selected Disclosures (HS-Disclosures)
   * `sd_disclosure` in II-Disclosures renamed to `sd_ii_disclosures`
   * `sd_disclosure` in HS-Disclosures renamed to `sd_hs_disclosures`
   * clarified relationship between `sd_hs_disclosure` and SD-JWT
   * clarified security requirements for blinded claim names
   * updated examples
   * text clarifications
   * fix `cnf` structure in examples
   * clarified that "alg=none" is allowed because when to check the signature is up to the Verifier's trust framework/policy, etc.) 

   -00

   * Upload as draft-ietf-oauth-selective-disclosure-jwt-00

   [[ pre Working Group Adoption: ]]

   -02

   *  Added acknowledgements
   *  Improved Security Considerations
   *  Stressed entropy requirements for salts
   *  Python reference implementation clean-up and refactoring
   *  `hash_alg` renamed to `sd_hash_alg`

   -01

   *  Editorial fixes
   *  Added `hash_alg` claim
   *  Renamed `_sd` to `sd_digests` and `sd_release`
   *  Added descriptions on holder binding - more work to do
   *  Clarify that signing the SD-JWT is mandatory

   -00

   *  Renamed to SD-JWT (focus on JWT instead of JWS since signature is optional)
   *  Make holder binding optional
   *  Rename proof to release, since when there is no signature, the term "proof" can be misleading
   *  Improved the structure of the description
   *  Described verification steps
   *  All examples generated from python demo implementation
   *  Examples for structured objects

