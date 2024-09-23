---
coding: utf-8

title: Bidding and Auction Services
docname: draft-ietf-bidding-and-auction-services-latest
category: std

area: TBD
workgroup: TBD
keyword: Internet-Draft

ipr: trust200902

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    name: Daniel Kocoj
    organization: Google
    email: dankocoj@google.com
 -
    ins: B. R. Hamilton
    name: Benjamin "Russ" Hamilton
    organization: Google
    email: behamilton@google.com

normative:
  CBOR: RFC8949
  CDDL: RFC8610
  JSON: RFC8259
  OHTTP: RFC9458
  SHA-256: RFC6234
  HPKE: RFC9180
  BINARY: RFC9292
  GZIP: RFC1952
  UUID: RFC9562
  ISO4217:
    target: https://www.iso.org/iso-4217-currency-codes.html
    title: ISO 4217 Currency codes
    date: 2024
  ORIGIN:
    target: https://html.spec.whatwg.org/multipage/webappapis.html#concept-settings-object-origin
    title: HTML Living Standard
    date: 2024
  URL:
    target: https://url.spec.whatwg.org/#concept-url
    title: URL Living Standard
    date: 2024
  ADRENDERID:
    target: https://wicg.github.io/turtledove/#server-auction-previous-win-ad-render-id
    title: Protected Audience
    date: 2024

informative:


--- abstract

The Bidding and Auction Services provide a way for advertising auctions to execute in
remote environments while preserving user privacy.

--- middle

# Introduction        {#problems}

Today, real-time bidding and ad auctions are executed on servers that may not
provide technical guarantees of security. Some users have concerns about how
their data is handled to generate relevant ads and in how that data is shared.
Protected Audience API ([Android](https://developer.android.com/design-for-safety/privacy-sandbox/fledge),
[Chrome](https://developer.chrome.com/docs/privacy-sandbox/fledge/)) provides
ways to preserve privacy and limit third-party data sharing by serving
personalized ads based on previous mobile app or web engagement.

For all platforms, Protected Audience may require [real-time services](https://github.com/privacysandbox/fledge-docs/blob/main/trusted_services_overview.md).
In the initial [proposal by Chrome](https://github.com/WICG/turtledove/blob/main/FLEDGE.md),
bidding and auction for Protected Audience ad demand is executed locally. This
can demand computation requirements that may be impractical to execute on
devices with limited processing power, or may be too slow to render ads due to
network latency.

This Bidding and Auction Services proposal outlines a way to allow Protected Audience
computation to take place on cloud servers,
rather than running locally on a user's device. Moving computations to
the cloud has the following benefits:

*   Scalable auctions.
  *   A scalable ad auction may include several buyers and sellers and that
      can demand more compute resources and network bandwidth.
*   System health of the user's device.
  *   Ensure better system health of the user's device by freeing up
      computational cycles and network bandwidth.
*   Better latency of ad auctions.
  *   Server to server communication on the cloud is faster than multiple device to server calls.
  *   Adtech code can execute faster on servers with higher computing power compared to a device.
*   Servers have better processing power.
  *   Adtechs can run more compute intensive workloads on a server compared to a device for better utility.

Standardized protocols for interacting with Bidding and Auction Services are
essential to creating a diverse and healthy ecosystem for such services.

## Scope {#Scope}

This document provides a specification for the request and response message
format that a client can use to communicate with remote services
that allows the client to offload much of the work involved in running an advertisement
selection auction as part of the client's implementation of the
Protected Audience API.

This document does not describe distribution of private keys to the Bidding
and Auction services.

## Terminology {#Terminology}

The key word "client" is to be interpreted as an implementation of this
document that creates Requests ({{client-to-services}}) and consumes Responses
({{services-to-client}}). The key phrase "Bidding and Auction Services" is
to be interpreted as an implementation of this document that consumes
Requests and creates Responses.

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL
NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED",
"MAY", and "OPTIONAL" in this document are to be interpreted as
described in BCP 14 {{!RFC2119}} {{!RFC8174}} when, and only when, they
appear in all capitals, as shown here.

# Message Format Specifications {#format}

## Overview

To understand this document, it is important to know that the
communication between the client and the remote services uses a
request-response message exchange pattern. The request will first reach a seller service, after which
the seller will forward parts of the request to buyer service. It is then up to the
seller service to gather buyer responses and form a final response for the client. More detail
about the seller and buyer services can be found in the [server-side system design documentation](https://github.com/privacysandbox/protected-auction-services-docs/blob/main/bidding_auction_services_system_design.md).

### Common Definitions {#common-definitions}

{{format}} makes frequent use of the following definitions.

| Term with CDDL Definition | Detailed Reference |
| :--- | :--- |
| `json = tstr` | [JSON] |
| `uuid = tstr .regexp "[a-fA-F0-9]{8}-[a-fA-F0-9]{4}-[a-fA-F0-9]{4}-[a-fA-F0-9]{4}-[a-fA-F0-9]{12}"` | [UUID] |
| `origin = tstr .regexp "https://([^/:](:[0-9]+)?/"` | [ORIGIN] |
| `currency = tstr .size 3 .regexp /^[A-Z]{3}$/` | [ISO4217] |
| `adRenderUrl = tstr` | [URL] |
| `adRenderId = tstr` | [ADRENDERID] |
| `interestGroupOwner = origin` | TODO |

### Message Framing and Padding {#framing}

Bidding and Auction Services requests and responses have the following framing:

| Byte     | 0         | 0             | 1 to 4   | 5 to Size+4       | Size+5 to end   |
| -------- | --------- | ------------- | -------- | ----------------- | --------------- |
| Bits     | 7-5       | 4-0           | *        | *                 | *               |
| -------- | --------- | ------------- | -------- | ----------------- | --------------- |
| Contents | Version   | Compression   | Size     | Request Payload   | Padding         |

where the the first 3 bits of the frame header specify the payload
version and the following 5 bits specify the compression algorithm.
The format described in this document corresponds to version 0.

The compression method's value in bits 4-0 in {{request-framing}}
corresponds to the below table:

| Compression | Description        |
| :---------: | :-------------     |
|      0      | No Compression     |
|      1      | Brotli {{!RFC7932}}|
|      2      | GZIP {{!RFC1952}}  |
|    3-31     | Reserved           |

The amount of padding depends on the type of message and will be discussed for
each message type separately.

## Request Format {#client-to-services}

This section discusses the request message sent from the client to the Bidding
and Auction Services endpoint.

The request from the Client to Bidding and Auction Services consists of an
[HPKE] encrypted payload with attached header (see {{request-encryption}}). The
plaintext payload contains a framing header, outer request, and padding (see
{{request-framing}}). The request message {{request-message}} is a [CBOR] encoded
message that contains one or more compressed interest group lists {{request-groups}}.

### Encryption {#request-encryption}

The request is encrypted with [HPKE]. The request uses a similar encapsulated message
format to that used by [OHTTP], with only an additional version field.

~~~~~
Encapsulated Request {
  Version (8),
  Key Identifier (8),
  HPKE KEM ID (16),
  HPKE KDF ID (16),
  HPKE AEAD ID (16),
  Encapsulated KEM Shared Secret (8 * Nenc),
  HPKE-Protected Request (..),
}
~~~~~

The `Version` for this message SHOULD be 0. The `HPKE KEM ID`, `HPKE KDF ID`,
and `HPKE AEAD ID` are the key encapsulation mechanism, key derivation function
and authenticated encryption with associated data function parameters for
[HPKE]. For this protocol, a compliant implementation MUST support
HKDF-SHA256 (0x0010) for `HPKE KEM ID`, HKDF-SHA256 (0x0001) for
`HPKE KDF ID`, and AES-256-GCM (0x0002) for `HPKE AEAD ID`.

Encryption of the request is similar to in [OHTTP]
[Section 4.3](https://www.rfc-editor.org/rfc/rfc9458#name-encapsulation-of-requests),
only with a different media type and the `Version` prepended to the encrypted
message:

1. Construct a message header (`hdr`) by concatenating the values of the
   `Key Identifier`, `HPKE KEM ID`, `HPKE KDF ID`, and `HPKE AEAD ID` in network
   byte order.
1. Build a sequence of bytes (`info`) by concatenating the ASCII-encoded string
   "message/auction request", a zero byte, and `hdr`.
1. Create a sending HPKE context by invoking `SetupBaseS()`
   ([Section 5.1.1](https://rfc-editor.org/rfc/rfc9180#section-5.1.1) of [HPKE])
   with the public key of the receiver `pkR` and `info`. This yields the context
   `sctxt` and an encapsulation key `enc`.
1. Encrypt `request` by invoking the `Seal()` method on `sctxt`
   ([Section 5.2](https://rfc-editor.org/rfc/rfc9180#section-5.2) of [HPKE])
   with empty associated data `aad`, yielding ciphertext `ct`.
1. Concatenate the values of `Version`, `hdr`, `enc`, and `ct`.

In pseudocode, this procedure is as follows:

~~~~~
hdr = concat(encode(1, key_id),
             encode(2, kem_id),
             encode(2, kdf_id),
             encode(2, aead_id))
info = concat(encode_str("message/auction request"),
              encode(1, 0),
              hdr)
enc, sctxt = SetupBaseS(pkR, info)
ct = sctxt.Seal("", request)
enc_request = concat(encode(1, version), hdr, enc, ct)
~~~~~

A Bidding and Auction Services endpoint decrypts this encapsulated message in a
similar manner to [OHTTP]
[Section 4.3](https://www.rfc-editor.org/rfc/rfc9458#name-encapsulation-of-requests),
or more explicitly as follows:

1. Parse `enc_request` into `version`, `key_id`, `kem_id`, `kdf_id`, `aead_id`,
   `enc`, and `ct`.
1. If `version` is not 0, return an error.
1. Find the matching HPTK private key, `skR`, corresponding to `key_id`. If there
   is no matching key, return an error.
1. Build a sequence of bytes (`info`) by concatenating the ASCII-encoded string
   "message/auction request"; a zero byte; `key_id` as an 8-bit integer; plus
   `kem_id`, `kdf_id`, and `aead_id` as three 16-bit integers.
1. Create a receiving HPKE context, `rctxt`, by invoking `SetupBaseR()`
   ([Section 5.1.1](https://rfc-editor.org/rfc/rfc9180#section-5.1.1) of [HPKE])
   with `skR`, `enc`, and `info`.
1. Decrypt `ct` by invoking the `Open()` method on `rctxt`
   ([Section 5.2](https://rfc-editor.org/rfc/rfc9180#section-5.2) of [HPKE]),
   with an empty associated data `aad`, yielding `request` and returning an
   error on failure.

In pseudocode, this procedure is as follows:

~~~~~
version, key_id, kem_id, kdf_id, aead_id, enc, ct = parse(enc_request)
if version != 0 then return error
info = concat(encode_str("message/auction request"),
              encode(1, 0),
              encode(1, key_id),
              encode(2, kem_id),
              encode(2, kdf_id),
              encode(2, aead_id))
rctxt = SetupBaseR(enc, skR, info)
request, error = rctxt.Open("", ct)
~~~~~

Bidding and Auction Services retains the HPKE context, `rctxt`, so that it can
encapsulate a response.

### Framing and Padding {#request-framing}

The plaintext message uses the framing described in {{framing}}.

Messages MAY be zero padded so that the encrypted request is one of the
following bin sizes: 0KiB, 5KiB, 10KiB, 20KiB, 30KiB, 40KiB, 55KiB. An
implementation MAY need to remove some data from the payload to fit inside the
largest bucket.

A compatible implementation processing requests SHOULD NOT rely on a specific
padding scheme for requests.

### Request Message {#request-message}

The request message is a [CBOR] encoded message with the following [CDDL] schema:

~~~~~ cddl
request = {
  // TODO description of all fields
  version: int,
  generationId: uuid,
  publisher: origin,
  interestGroups: {
    ; Map of interest group owner to CBOR encoded list of interest
    ; groups compressed using the method described in § Compression.
    * interestGroupOwner => bstr
  },
  ? enableDebugReporting: bool
}
~~~~~

The `version` field SHOULD be set to 0 for this version of the protocol. The
`interestGroups` field is a map from interest group owner to a compressed,
[CBOR]-encoded list of interest groups for that owner.

#### List of Interest Groups {#request-groups}

A list of interest group is encoded in [CBOR] with the following [CDDL] schema:

~~~~~ cddl
interestGroups = [ * interestGroup ]
interestGroup = {
  ; This interest group's name, see
  ; https://wicg.github.io/turtledove/#interest-group-name.
  name: tstr,

  ; Keys used to look up real-time bidding signals, see
  ; https://wicg.github.io/turtledove/#interest-group-trusted-bidding-signals-keys.
  ? biddingSignalsKeys: [* tstr],
  ; Data about the user that the bidder can use during bid calculation, see
  ; https://wicg.github.io/turtledove/#interest-group-user-bidding-signals.
  ? userBiddingSignals: json,
  ; Contains various ads that the interest group might show. See
  ; https://wicg.github.io/turtledove/#interest-group-ads.
  ? ads: [* adRenderId],

  ; Contains various ad components (or "products") that can be used to
  ; construct ads composed of multiple pieces — a top-level ad template
  ; "container" which includes some slots that can be filled in with
  ; specific "products". See
  ; https://wicg.github.io/turtledove/#interest-group-ad-components.
  ? components: [* adRenderId],
  ? browserSignals: {
    ; Number of times the group was joined in the last 30 days.
    ? joinCount: int,

    ; Number of times the group bid in an auction in the last 30
    ; days.
    ? bidCount: int,

    ; Tuple of time-ad pairs for a previous win for this interest
    ; group that occurred in the last 30 days.
    ; The time is specified in seconds before the containing
    ; auctionBlob was requested.
    ? prevWins: [* [int, adRenderId]],

    ; The most recent join time for this group expressed
    ; in milli seconds before the containing auctionBlob
    ; was requested. This field will be used by newer client
    ; versions. For older devices, the precison will be in seconds.
    ; If recencyMs is present, this value will be used to offer
    ; higher precision. If not, recency will be used. Only
    ; one of the recency or recencyMs is expected to present in
    ; the request.
    ? recencyMs: int
  }
}
~~~~~

Each list is separately compressed with the compression method indicated in
the {{framing}} header.

### Generating a Request {#request-generate}

This section describes how the client MAY form and serialize request messages
in order to communicate with the Bidding and Auction services.

This algorithm takes as input the `publisher`, all of the `relevant interest groups`,
an optional `desired total size`, an optional list of `interest group owners` to
include each with an optional `desired size`, and the [HPKE] `public key` and
its associated `key ID`. It returns an `encrypted request` and a `request context`
tuple.

1. Let `included_groups` be an empty map.
1. If `desired total size` is not specified, but the list of `interest group owners`
   includes at least one entry with a specified `desired size`:
   1. Set `desired total size` to the sum of all specified `desired size` in the
      list of `interest group owners`.
1. Group the list of `relevant interest groups` by owner into a map of from
   interest group owner to a list of interest groups sorted by decreasing
   priority, `interest group map`.
1. If the list of `interest group owners` is specified, remove interest groups
   whose owner is not on the list.
1. Construct a request, `request` with `request["publisher"]` set to `publisher`,
   `request["version"]` set to 0, `request["generationId"]` set to a new [UUID]
   [Version 4](https://www.rfc-editor.org/rfc/rfc9562.html#section-5.4),
   and `request["enableDebugReporting"]` set to true.
1. Set `current_size` to be the serialized size of the encrypted request
   created from `request` without padding.
1. Set `remaining_allocated_size` to 0.
1. Set `remaining_unsized_owners` to 0.
1. For each `interest group owner`, `interest group list` in
   `interest group map`:
   1. If there is a `desired size` for `interest group owner`:
      1. Increment `remaining_allocated_size` by `desired size`.
   1. Otherwise
      1. Increment `remaining_unsized_owners` by 1.
1. For each `interest group owner`, `interest group list` in
   `interest group map` where there is a `desired size` specified for
   `interest group owner`:
   1. If the number of `unsized_owners` is not 0:
      1. Set the `allowed_interest_group_size` to the `desired size` for this
         `interest group owner`. This is a fixed size allocation.
   1. Otherwise:
      1. Let `remaining_size` be equal to the `desired total size`-`current_size`.
      1. Set the `allowed_interest_group_size` to
         `remaining_size`*`desired_size`/`remaining_allocated_size`. This is a
         proportional allocation.
   1. Set `remaining_allocated_size` = `remaining_allocated_size`-`current_size`.
   1. [CBOR] encode the `interest group list` into `serialized list`.
   1. [GZIP] the `serialized list` into `compressed list`.
   1. If setting `request["interestGroups"][interest group owner]` to
      `compressed list` would make it's serialized size more than
      `allowed_interest_group_size` larger than the current size, then remove
      the lowest priority interest group and repeat from the previous step.
   1. Set `request["interestGroups"][interest group owner]` to
      `compressed list`.
   1. Set `included_groups[interest group owner]` to `interest group list`.
   1. Set `current_size` to be the serialized size of the encrypted request
      created from `request` without padding.
1. For each `interest group owner`, `interest group list` in
   `interest group map` where there is not `desired size` specified for
   `interest group owner`:
   1. Let `remaining_size` be equal to the `desired total size`-`current_size`.
   1. Set the `allowed_interest_group_size` to
      `remaining_size`*/`remaining_unsized_owners`. This is a
      equal size allocation.
   1. Decrement `remaining_unsized_owners` by 1.
   1. [CBOR] encode the `interest group list` into `serialized list`.
   1. [GZIP] the `serialized list` into `compressed list`.
   1. If adding the `compressed list` to `request` would make it more than
      `allowed_interest_group_size` larger than the current size, then remove
      the lowest priority interest group and repeat from the previous step.
   1. Set `request["interestGroups"][interest group owner]` to
      `compressed list`.
   1. Set `included_groups[interest group owner]` to `interest group list`.
   1. Set `current_size` to be the serialized size of the encrypted request
      created from `request` without padding.
1. If there are no interest groups in the request, discard the `request` and
   return failure.
1. Prepend the framing header to `request` with `Compression` set to 2.
1. If `desired total size` is set then zero pad `request` to `desired total size`.
   Otherwise zero pad `request` up to the smallest bin size in {{request-framing}}
   larger than request.
1. Encrypt `request` using the `public key` and its `key id` as in
   {{request-encryption}} to get the `encrypted message` and `hpke context`.
1. Let the `request context` be the tuple (`included_groups`, `hpke context`)
1. Return the `request` and the `request context`.

### Parsing a Request {#request-parsing}

This algorithm takes as input an `encrypted request` and an [HPKE] `private key`.

## Response Format {#services-to-client}

This section discusses the request message sent from the Bidding and Auction
Services endpoint to the client in reply to a request.

The response from the Bidding and Auction Services endpoint consists of an
[HPKE] encrypted payload with attached header (see {{response-encryption}}). The
plaintext payload contains a framing header, response message, and padding (see
{{response-framing}}). The response message {{response-message}} is a compressed
[CBOR] encoded message.

### Encryption {#response-encryption}

The response is encrypted with [HPKE] as a

The response uses a similar encapsulated response format to that used by
[OHTTP].

~~~~~
Encapsulated Response {
  Nonce (8 * max(Nn, Nk)),
  AEAD-Protected Response (..),
}
~~~~~

Encryption of the response is similar to in [OHTTP] [Section 4.4](https://www.rfc-editor.org/rfc/rfc9458.html#name-encapsulation-of-responses), only with a different media type,
repeated below for clarity:

1. Export a secret (`secret`) from `context`, using the string
   "message/auction response" as the `exporter_context` parameter to
   `context.Export`; see [Section 5.3](https://www.rfc-editor.org/rfc/rfc9180.html#name-secret-export)
   of [HPKE]. The length of this secret is `max(Nn, Nk)`, where `Nn` and `Nk` are
   the length of the AEAD key and nonce that are associated with `context`.
1. Generate a random value of length `max(Nn, Nk)` bytes, called `response_nonce`.
1. Extract a pseudorandom key (`prk`) using the `Extract` function provided by
   the KDF algorithm associated with context. The `ikm` input to this function
   is `secret`; the `salt` input is the concatenation of `enc` (from
  `enc_request`) and `response_nonce`.
1. Use the `Expand` function provided by the same KDF to create an AEAD key,
   `key`, of length `Nk` -- the length of the keys used by the AEAD associated
   with `context`. Generating `aead_key` uses a label of "key".
1. Use the same `Expand` function to create a nonce, `nonce`, of length `Nn`
   -- the length of the nonce used by the AEAD. Generating `aead_nonce` uses a
  label of "nonce".
1. Encrypt `response`, passing the AEAD function `Seal` the values of `aead_key`,
   `aead_nonce`, an empty `aad`, and a `pt` input of `response`. This yields `ct`.
1. Concatenate `response_nonce` and `ct`, yielding an Encapsulated Response,
   `enc_response`. Note that `response_nonce` is of fixed length, so there is no
  ambiguity in parsing either `response_nonce` or `ct`.

In pseudocode, this procedure is as follows:

~~~~~
secret = context.Export("message/auction response", max(Nn, Nk))
response_nonce = random(max(Nn, Nk))
salt = concat(enc, response_nonce)
prk = Extract(salt, secret)
aead_key = Expand(prk, "key", Nk)
aead_nonce = Expand(prk, "nonce", Nn)
ct = Seal(aead_key, aead_nonce, "", response)
enc_response = concat(response_nonce, ct)
~~~~~

Clients decrypt an Encapsulated Response by reversing this process. That is,
Clients first parse `enc_response` into `response_nonce` and `ct`. Then, they
follow the same process to derive values for `aead_key` and `aead_nonce`, using
their sending HPKE context, `sctxt`, as the HPKE context, `context`.

The Client uses these values to decrypt `ct` using the AEAD function `Open`.
Decrypting might produce an error, as follows:

~~~~~
response, error = Open(aead_key, aead_nonce, "", ct)
~~~~~

### Framing and Padding {#response-framing}

The plaintext message uses the framing described in {{framing}}.

Messages MAY be exponentially padded so that the encrypted response is a power
of 2 in length.

A compatible implementation processing requests SHOULD NOT rely on a specific
padding scheme for requests.


### Response Message {#response-message}

The response message is a [CBOR] encoded message, compressed using the method
indicated in {#request-framing}. The [CBOR] encoded message SHOULD be
serialized into deterministically encoded [CBOR] (as defined in
[Section 4.2](https://www.rfc-editor.org/rfc/rfc8949.html#name-deterministically-encoded-c))
and follows the following [CDDL] schema:

~~~~~cddl
response = {
  ; The ad to render.
  ; Maps directly to https://wicg.github.io/turtledove/#server-auction-response-ad-render-url.
  adRenderURL: adRenderUrl,

  ; List of URLs for component ads displayed as part of this
  ; ad.
  ; Maps directly to https://wicg.github.io/turtledove/#server-auction-response-ad-components.
  ; If not present, map as an empty list.
  ? components: [* adRenderUrl],

  ; Name of the interest group to which the ad belongs.
  ; See https://wicg.github.io/turtledove/#interest-group-name.
  ; Maps directly to https://wicg.github.io/turtledove/#server-auction-response-interest-group-name.
  ; If not present, map as Null.
  ? interestGroupName: tstr,

  ; Origin of the Buyer who owns the interest group.
  ; The original request for this response MUST contain this
  ; interestGroupOwner, which additionally MUST provide an interest
  ; group with interestGroupName.
  ; Maps directly to https://wicg.github.io/turtledove/#server-auction-response-interest-group-owner.
  ; If not present, map as Null.
  ? interestGroupOwner: interestGroupOwner,

  ; Indices of interest groups in the original request for this owner
  ; that submitted a bid.
  ; Maps to https://wicg.github.io/turtledove/#server-auction-response-bidding-groups.
  ; If not present, map as an empty list.
  ; Else,
  ;   1. create an empty list
  ;   2. for each interest group owner key in the biddingGroups map
  ;   3.    for each index in biddingGroups[interest group owner]
  ;   4.       interest group name equals the string at the index from (3)
  ;            in Encryption Context's Interest Group Map (Section 2.2.4.1.2)
  ;   5.       add a tuple to the list in (1) of [interest group owner (2),
               interest group name (4)]
  ;   6. return the list in (1)
   ? biddingGroups: {
    * interestGroupOwner => [* int]
  },

  ; Score of the ad determined during the auction.
  ; Any value that is zero or negative indicates that the ad cannot
  ; win the auction.
  ; The winner of the auction would be the ad that was given the
  ; highest score.
  ; Maps directly to https://wicg.github.io/turtledove/#server-auction-response-score.
  ; If not present, map as Null.
  ? score: float,

  ; Bid price corresponding to an ad
  ; Maps directly to https://wicg.github.io/turtledove/#server-auction-response-bid.
  ; If not present, map as Null.
  ? bid: float,

  ; Optional currency of the bid.
  ; Maps directly to https://wicg.github.io/turtledove/#server-auction-response-bid-currency.
  ; If not present, map as Null.
  ? bidCurrency: currency,

  ; Optional BuyerReportingId of the winning Ad
  ; Maps directly to https://wicg.github.io/turtledove/#server-auction-response-buyer-reporting-id.
  ; If not present, map as Null.
  ? buyerReportingId: tstr,

  ; Optional BuyerAndSellerReportingId of the winning Ad
  ; Maps directly to https://wicg.github.io/turtledove/#server-auction-response-buyer-and-seller-reporting-id.
  ; If not present, map as Null.
  ? buyerAndSellerReportingId: tstr,

  ; The auction result may be ignored if set to true.
  ; Maps to https://wicg.github.io/turtledove/#server-auction-response-is-chaff.
  ; If not present, map as false.
  ? isChaff: bool,

  ; Optional wrapper for various reporting URLs.
  ? winReportingUrls: {
    ; Maps to https://wicg.github.io/turtledove/#server-auction-response-buyer-reporting.
    ; If not present, map as 'Null'.
    ? buyerReportingUrls: reportingUrls,
    ; Maps to https://wicg.github.io/turtledove/#server-auction-response-component-seller-reporting.
    ; If not present, map as 'Null'.
    ? componentSellerReportingUrls: reportingUrls,
    ; Maps to https://wicg.github.io/turtledove/#server-auction-response-top-level-seller-reporting.
    ; If not present, map as 'Null'.
    ? topLevelSellerReportingUrls: reportingUrls
  },

  ; Contains an error message from the auction executed on the trusted auction
  ; server.
  ; May be used to provide additional context for the result of an auction.
  ; Maps to https://wicg.github.io/turtledove/#server-auction-response-error.
  ; If not present, map as Null.
  ; Else, ignore the `code` field and use the `message` field directly
  ; for the server auction response error field.
  ? error: {
    code: int,
    message: tstr
  },

  ; Arbitrary metadata to pass to the top-level seller.
  ; Maps directly to https://wicg.github.io/turtledove/#server-auction-response-ad-metadata.
  ; If not present, map as Null.
  ? adMetadata: json,

  ; Optional name/domain for the top-level seller in case this is a
  ; component auction.
  ; Maps directly to https://wicg.github.io/turtledove/#server-auction-response-top-level-seller.
  ; If not present, map as Null.
  ? topLevelSeller: origin,
}

; Defines the structure for reporting URLs.
reportingUrls = {
  ; Maps directly to https://wicg.github.io/turtledove/#server-auction-reporting-info-reporting-url.
  ; If not present, map as Null.
  ? reportingUrl: tstr,
  ; Maps directly to https://wicg.github.io/turtledove/#server-auction-reporting-info-beacon-urls.
  ; If not present, map as an empty ordered map (https://infra.spec.whatwg.org/#ordered-map).
  ? interactionReportingUrls: { * tstr => tstr }
}

~~~~~

### Generating a Response {#response-generate}

TODO

### Parsing a Response {#response-parse}

This algorithm describes how a conforming Client MUST parse and validate a
response from Bidding and Auction Services. It takes as input the
`request context` tuple returned from {{request-generate}} in addition to the
`encrypted response`.

1. Use `request context`'s `hpke context` as the `context` for decryption and
   follow the decryption steps in {{response-encryption}} to decrypt
   `encrypted response` and obtain `framed response` and `error`.
1. If `error` is not null, return failure.
1. Remove and extract the first 5 bytes from `framed response` as the
   `framing header` (described in {{framing}}), removing them from
   `framed response`.
1. If the `framing header`'s `Version` field is not 0, return failure.
1. Let `length` be equal to the `framing header`'s `Size` field.
1. If `length` is greater than the length of the remaining bytes in
   `framed response`, return failure.
1. Take the first `length` remaining bytes in `framed response` as
   `compressed response`, discarding the rest.
1. Decompress the `compressed response` into `serialized response` using the
   method indicated by `framing header`'s `Compression` field, returning
   failure if decompression fails.
1. [CBOR] decode the `serialized response` into `response`, returning failure
   if decompression fails.
1. If `response` is not a map, return failure.
1. If `response["error"]` exists, return failure.
1. If `response["isChaff"]` exists and is either not a boolean or is true,
   return failure.
1. Let `processed response` be a new structure analogous to
   [server auction response](https://wicg.github.io/turtledove/#server-auction-response).
1. If `response["adRenderURL"]` does not exist, return failure.
1. Set `processed response["ad render url"]` to `response["adRenderURL"]` parsed
   as a [URL], returning failure if there is an error.
1. If `response["components"]` exists:
  1. If `response["components"]` is not an array, return failure.
  1. For each `component` in `response["components"]`:
    1. Append `component` parsed as a [URL] to
       `processed response["ad components"]`, returning failure if there is an
       error.
1. If `response["interestGroupName"]` does not exist or is not a string, return failure.
1. Set `processed response["interest group name"]` to `response["interestGroupName"]`.
1. If `response["interestGroupOwner"]` does not exist or is not a string, return failure.
1. Set `processed response["interest group owner"]` to `response["interestGroupOwner"]`
   parsed as an [ORIGIN], returning failure if there is an error.
1. If `response["biddingGroups"]` does not exist or is not a map, return failure.
1. For each `key`, `value` in `response["biddingGroups"]`:
  1. Let `owner` be equal to `key` parsed as an [ORIGIN], returning failure if
     there is an error.
  1. `request context`'s `included_groups` does not contain `owner` as a key, return failure.
  1. If `value` is not a list, return failure.
  1. For each `element` in `value`:
    1. If `element` is not an integer or `element < 0`, return failure.
    1. If `element` is greater than or equal to the length of
       `included_groups[owner]`, return failure.
    1. Let `name` be the `interest group name` for `included_groups[owner][element]`.
    1. Append the tuple (`owner`, `name`) to `processed response["bidding groups"]`.
1. If `response["score"]` exists:
  1. If `response["score"]` is not a floating point value, return failure.
  1. Set `processed response["score"]` to `response["score"]`.
1. If `response["bid"]` exists:
  1. If `response["bid"]` is not a floating point value, return failure.
  1. Let `bid` be a new structure analogous to [bid with currency](https://wicg.github.io/turtledove/#bid-with-currency).
  1. Set `bid`s `value` field to `response["bid"]`.
  1. If `response["bidCurrency"]` exists:
    1. If `response["bidCurrency"]` is not a string, return failure.
    1. If `response["bidCUrrency"]` is not 3 bytes long or contains characters
       other than upper case ASCII letters, return failure.
    1. Set `bid`'s `currency` field to `response["bidCurrency"]`.
  1. Set `processed response["bid"]` to `bid`.
1. If `response["winReportingURLs"]` exists and is a map:
  1. If `response["winReportingURLs"]["buyerReportingURLs"]` exists:
    1. Let `buyer reporting` be the result of {{response-parsing-reporting}} on
       `response["winReportingURLs"]["buyerReportingURLs"]`.
    1. Set `processed response["buyer reporting"]` to `buyer reporting`.
  1. If `response["winReportingURLs"]["topLevelSellerReportingURLs"]` exists:
    1. Let `top level seller reporting` be the result of {{response-parsing-reporting}}
       on  `response["winReportingURLs"]["topLevelSellerReportingURLs"]`.
    1. Set `processed response["top level seller reporting"]` to
       `top level seller reporting`.
  1. If `response["winReportingURLs"]["componentSellerReportingURLs"]` exists:
    1. Let `component seller reporting` be the result of {{response-parsing-reporting}}
       on `response["winReportingURLs"]["componentSellerReportingURLs"]`.
    1. Set `processed response["component seller reporting"` to
       `component seller reporting`.
1. If `response["topLevelSeller"]` exists:
  1. If `response["topLevelSeller"]` is not a string, return failure.
  1. Set `processed response["top level seller"]` to `response["topLevelSeller"]`
     parsed as a [URL], returning failure if there is an error.
1. If `response["adMetadata"]` exists and is a string set
   `processed response["ad metadata"]` to `response["adMetadata"]`.
1. If `response["buyerReportingId"]` exists and is a string set
   `processed response["buyer reporting id"]` to `response["buyerReportingId"]`.
1. If `response["buyerAndSellerReportingId"]` exists and is a string set
   `processed response["buyer and seller reporting id"]` to
   `response["buyerAndSellerReportingId"]`.
1. Return `processed response`.

#### Parsing reporting URLs {#response-parsing-reporting}

To parse reporting URLs on a [CBOR] map `reporting URLs` with a schema like
`reportingUrls` from {{response-message}}:

1. Let `processed reporting URLs` be a new structure analogous to
   [server auction reporting info](https://wicg.github.io/turtledove/#server-auction-reporting-info).
1. If `reporting URLs["reportingURL"]` exists and is a string:
  1. Let `reporting URL` be `reporting URLs["reportingURL"]` parsed as a [URL],
     or null if there is an error.
  1. If `reporting URL` is not null, set
     `processed reporting URLs["reporting url"]` to `reporting URL`.
1. If `reporting URLs["interactionReportingURLs"]` exists and is a map:
  1. For each `key`, `value` in `reporting URLs["interactionReportingURLs"]`:
    1. If `key` is not a string, continue with the next iteration.
    1. Let `reporting URL` be `value` parsed as a [URL]. If there is an error,
       continue with the next iteration.
    1. Set `processed reporting URLs["beacon urls"][key]` to `reporting URL`.
1. Return `processed reporting URLs`.

# Security Considerations

TODO

# IANA Considerations

This document introduces no additional considerations for IANA.

--- back

# Acknowledgments
{:numbered="false"}

TODO
