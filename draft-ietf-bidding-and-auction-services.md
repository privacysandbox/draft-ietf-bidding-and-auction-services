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
  HPKE: RFC9180
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
  IGOWNER:
    target: https://wicg.github.io/turtledove/#server-auction-response-interest-group-owner
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

This Bidding and Auction Services proposal outlines a way to allow Protected Audience
computation to take place on cloud servers in a
Trusted Execution Environment (TEE),
rather than running locally on a user's device. Running workloads
in a TEE in cloud has the following benefits:

*   Scalable ad auctions.
    *   A scalable ad auction may include several buyers and sellers and that
        can demand more compute resources and network bandwidth.
*   Lower latency of ad auctions.
    *   Server to server communication on the cloud is faster than multiple
        device to server calls.
    *   Adtech code can execute faster on servers with higher computing power.
*   Higher utility of ad auctions.
    *   Servers have better processing power, therefore adtechs can run compute
        intensive workloads on a server for better utility.
    *   Lower latency of ad auctions also positively impact utility.
*   Security protection
    *   TEEs can protect confidentiality of adtech code and signals.
*   System health of the user's device.
    *   Ensure better system health of user's device by freeing up computational
        cycles and network bandwidth.

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
| `interestGroupOwner = origin` | [IGOWNER] |

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

GZIP MUST be implemented by the Client and the Bidding and Auction Services,
while Brotli MAY be.

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
DHKEM(X25519, HKDF-SHA256) (0x0020) for `HPKE KEM ID`, HKDF-SHA256 (0x0001) for
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
1. Find the matching HPKE private key, `skR`, corresponding to `key_id`. If
   there is no matching key, return an error.
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
  ; Current version of the protocol.
  ; In this document, it must be 0.
  version: int,
  ; Used by the Bidding and Auction Services to
  ; keep track of a request (and corresponding response)
  ; over its lifetime.
  ; Must be a [UUID][Version 4].
  generationId: uuid,
  ; Represents the publisher initiating the request.
  publisher: origin,
  interestGroups: {
    ; Map of interest group owner to CBOR encoded list of interest
    ; groups compressed as described in § Generating a Request.
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

This algorithm takes as input all of the `relevant interest groups`, a config
consisting of the `publisher`, a map of from (origin, string) tuple to origin
`ig pagg coordinators`,
an optional `desired total size`, an optional boolean `debugging report locked out`
defaults to false, an optional list of `interest group owners` to
include each with an optional `desired size`, and the [HPKE] `public key` with
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
   and `request["enableDebugReporting"]` set to `debugging report locked out`.
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
1. Let the `request context` be the tuple
   (`included_groups`, `hpke context`, `ig pagg coordinators`).
1. Return the `encrypted message` and the `request context`.

### Parsing a Request {#request-parsing}

This section describes how the Bidding and Auction Services MUST deserialize
request messages from the client.

The algorithm takes as input a serialized request message from the client
({{request-generate}}) and a list of HPKE private keys (along with their
corresponding key IDs).

The output is either an error sent back to the client, an empty message sent
back to the client, or a request message the Bidding and Auction services can
consume along with an HPKE context.

1. Let `encrypted request` be the request received from the client.
1. Let `error_msg` be an empty string.
1. De-encapsulate and decrypt `encrypted request` by using the input private key
   corresponding to `key_id`, as described in {{request-encryption}}, to get the
   decrypted message and `rctxt`.
   1. If decapsulation or decryption fails, return failure.
   1. Else, save the decrypted output as `framed request` and save `rctxt`.
1. Remove and extract the first 5 bytes from `framed request` as the
   `framing header` (described in {{framing}}), removing them from
   `framed request`.
1. If the `framing header`'s `Version` field is not 0, return failure.
1. If the `framing header`'s `Compression` field is not supported, return
   failure. Otherwise, save the `Compression` field value as `compression type`.
1. Let `length` be equal to the `framing header`'s `Size` field.
1. If `length` is greater than the length of the remaining bytes in
   `framed request`, return failure.
1. Take the first `length` remaining bytes in `framed response` as
   `decodable request`, discarding the rest.
1. [CBOR] decode `decodable request` into the message represented in {{request-message}}.
   Let this be `request`.
1. If [CBOR] decoding fails, return failure.
1. Let `processed request` be an empty struct.
1. If `request` is not a map, return failure.
1. If `request["version"]` does not exist or is not 0, return failure.
1. If `request["publisher"]` does not exist or is not a string, return failure.
1. Set `processed request["publisher"]` to `request["publisher"]`.
1. If `request["generationId"]` does not exist or is not a string, return failure.
1. Set `processed request["generationId"]` to `request["generationId"]`.
1. If `request["enableDebugReporting]` exists:
   1. If `request["enableDebugReporting"]` is not a boolean, return failure.
   1. Set `processed request["enableDebugReporting"]` to
      `request["enableDebugReporting"]`.
1. If `request["interestGroups]` does not exist or is not a map, return failure.
1. Set `processed request["interestGroups"]` to an empty map.
1. For each `key`, `value` map entry of `request["interestGroups"]`:
   1. If `key` is not a string, append an error message to
      `error_msg`. Proceed to {{request-parse-error}}.
   1. Set `processed request["interestGroups"]``[key]` to an empty list.
   1. Decompress `value` according to `compression type` and set as
      `buyer input cbor`. If decompression fails, return failure.
   1. [CBOR] decode `buyer input cbor` into `buyer input`. If decoding fails, return failure.
   1. If `buyer input` is not an array, return failure.
   1. For each `interest group` in `buyer input`:
      1. If the `interest groups` is not a map, append an error message to
         `error_msg`. Proceed to {{request-parse-error}}.
      1. Let `ig` be an empty struct similar to {{request-groups}}.
      1. If `interest group["name"]` does not exist or is not a string, return failure.
      1. Set `ig["name"]` to `interest group["name"]`.
      1. If `interest group["userBiddingSignals"]` exists:
         1. If `interest group["userBiddingSignals"]` is not a string, return failure.
         1. Set `ig["userBiddingSignals"]` to `interest group["userBiddingSignals"]`.
      1. If `interest group["biddingSignalsKeys"]` exists:
         1. If `interest group["biddingSignalsKeys"]` is not an array of strings,
            return failure.
         1. Set `ig["biddingSignalsKeys"]` to `interest group["biddingSignalsKeys"]`.
      1. If `interest group["ads"]` exists:
         1. If `interest group["ads"]` is not an array of strings, return failure.
         1. Set `ig["ads"]` to `interest group["ads"]`.
      1. If `interest group["component"]` exists:
         1. If `interest group["component"]` is not an array
         of strings, return failure.
         1. Set `ig["component"]` to `interest group["component"]`.
      1. If `interest group["browserSignals"]` exists:
         1. If `interest group["browserSignals"]` is not a map, return failure.
         1. Let `igbs` be an empty struct similar to `browserSignals` as
            defined in {{request-groups}}.
         1. Let `signals` be `interest group["browserSignals"]`.
         1. If `signals["bidCount"]` exists:
            1. If `signals["bidCount"]` is not a valid 64-bit unsigned integer,
               return failure.
            1. Set `igbs["bidCount"]` to `signals["bidCount"]`.
         1. If `signals["joinCount"]` exists:
            1. If `signals["joinCount"]` is not a valid 64-bit unsigned integer,
               return failure.
            1. Set `igbs["joinCount"]` to `signals["joinCount"]`.
         1. If `signals["recencyMs"]` exists:
            1. If `signals["recencyMs"]` is not a valid 64-bit unsigned integer,
               return failure.
            1. Set `igbs["recencyMs"]` to `signals["recencyMs"]`.
         1. If `signals["prevWins"]` exists:
            1. Let `pw` be an empty array.
            1. If `signals["prevWins"]` is not an array, return failure.
            1. For each `prevWinTuple` in `signals["prevWins"]`:
               1. Let `pwt` be an empty array.
               1. If `prevWinTuple` is not an array of size 2, return failure.
               1. If `prevWinTuple[0]` is not a valid 64-bit unsigned integer,
                  return failure.
               1. If `prevWinTuple[1]` is not a string, return failure.
               1. Set `pwt` to `prevWinTuple`.
               1. Append `pwt` to `pw`.
            1. Set `igbs["prevWins"]` to `pw`.
         1. Set `ig["browserSignals"]` to `igbs`.
      1. Append `ig` to `processed request["interestGroups"]``[ key ]`.
1. Return `processed request` and `rctxt` to the Bidding and Auction
   Services.

#### Request Parse Error Handling {#request-parse-error}

If {{request-parsing}} returns with failure, the following algorithm describes
how the Bidding and Auction Services MUST respond.

The input to this algorithm is `error_msg`, which MAY be returned from the point
of failure in {{request-parsing}}.

The output is a response to the client.

1. If the failure happens before or during decryption, respond with an empty
   message.
1. Otherwise abort processing the request.
1. Let `error` be a new map with key-value pairs:
   `[("code", 400), ("error", error_msg)]`.
1. Let `response` be a new {{response-message}}.
1. Set `response["error"]` to `error`.
1. Serialize and send `response` to the client per {{services-to-client}}.

## Response Format {#services-to-client}

This section discusses the request message sent from the Bidding and Auction
Services endpoint to the client in reply to a request.

The response from the Bidding and Auction Services endpoint consists of an
[HPKE] encrypted payload with attached header (see {{response-encryption}}). The
plaintext payload contains a framing header, response message, and padding (see
{{response-framing}}). The response message {{response-message}} is a compressed
[CBOR] encoded message.

### Encryption {#response-encryption}

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
  ; An unguessable nonce to use when authorizing this response.
  ? nonce: tstr,

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

  ; Indices and update-if-older-than times of interest groups in the original
  ; request for this owner for interest groups where an update-if-older-than
  ; time (in milliseconds) was specified.
  ; Maps to https://wicg.github.io/turtledove/#server-auction-response-update-groups.
  ? updateGroups: {
    * interestGroupOwner => [
      * {
        index: int,
        updateIfOlderThanMs: int
      }]
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

  ; Optional SelectedBuyerAndSellerReportingId of the winning Ad
  ; Maps directly to https://wicg.github.io/turtledove/#server-auction-response-selected-buyer-and-seller-reporting-id
  ; If not present, map as Null.
  ? selectedBuyerAndSellerReportingId: tstr,

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

  ; Optional list of forDebuggingOnly reports.
  ; If not present, map as an empty list.
  ; Maps to https://wicg.github.io/turtledove/#server-auction-response-component-win-debugging-only-reports
  ; and https://wicg.github.io/turtledove/#server-auction-response-server-filtered-debugging-only-reports.
  ? debugReports: [
    * {
      origin => [
        * {
          url tstr,
          ? bool isWinReport,
          ? bool isSellerReport,
          ? bool componentWin

  ; Optional list of private aggregation contributions.
  ; If not present, map as an empty list.
  ? paggResponse: [
    * {
      origin => [
        * {
          ? igIndex: int,
          ? coordinator: origin,
          ? componentWin: bool,
          eventContributions: [
            tstr => [
              * {
                bucket: blob,
                value: int
              }
            ]
          ]
        }
      ]
    }
  ],
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

This algorithm describes how conforming Bidding and Auction Services MAY
generate a response to a request.

The input is a `payload` corresponding to {{response-message}} and the HPKE
receiver context saved in {{request-parsing}}, `rctxt`.

The output is a `response` to be sent to a Client.

1. Let `cbor payload` equal the [deterministically encoded CBOR](https://www.rfc-editor.org/rfc/rfc8949.html#name-deterministically-encoded-c)
   `payload`. Return an empty `response` on CBOR encoding failure.
1. Let `compressed payload` equal the [GZIP] compressed `cbor payload`,
   returning an empty `response` on compression failure.
1. Create a framed payload, as described in {{response-framing}}:
   1. Create a `framing header`.
   1. Set the `framing header` `Compression` to 2.
   1. Set the `framing header` `Version` to 0.
   1. Set the `framing header` `Size` to the size of `compressed payload`.
   1. Let `framed payload` equal the result of prepend the framing header
      to `compressed payload`.
   1. Padding MAY be added to `framing header`, as described in
      {{response-framing}}.
   1. Return an empty `response` on failure of any of the previous steps.
1. Let `response` equal the result of the encryption and encapsulation of
   `framed payload` with `rctxt`, as described in {{response-encryption}}.
   Return an empty `response` on failure.
1. Return `response`.

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
1. If `response["nonce"]` exists and is a valid [UUID], set
   `processed response["nonce"]` to the lowercase ASCII representation of `response["nonce"]`.
1. If `response["adRenderURL"]` does not exist, return failure.
1. Set `processed response["ad render url"]` to `response["adRenderURL"]` parsed
   as a [URL], returning failure if there is an error.
1. If `response["components"]` exists:
   1. If `response["components"]` is not an array, return failure.
   1. For each `component` in `response["components"]`:
      1. Append `component` parsed as a [URL] to `processed response["ad components"]`,
         returning failure if there is an error.
1. If `response["interestGroupName"]` does not exist or is not a string, return failure.
1. Set `processed response["interest group name"]` to `response["interestGroupName"]`.
1. If `response["interestGroupOwner"]` does not exist or is not a string, return failure.
1. Set `processed response["interest group owner"]` to `response["interestGroupOwner"]`
   parsed as an [ORIGIN], returning failure if there is an error.
1. If `response["biddingGroups"]` does not exist or is not a map, return failure.
1. For each `key`, `value` in `response["biddingGroups"]`:
   1. Let `owner` be equal to `key` parsed as an [ORIGIN], returning failure if
      there is an error.
   1. If `request context`'s `included_groups` does not contain `owner` as a key, return failure.
   1. If `value` is not a list, return failure.
   1. For each `element` in `value`:
      1. If `element` is not an integer or `element < 0`, return failure.
      1. If `element` is greater than or equal to the length of
         `included_groups[owner]`, return failure.
      1. Let `name` be the `interest group name` for `included_groups[owner][element]`.
      1. Append the tuple (`owner`, `name`) to `processed response["bidding groups"]`.
1. If `response["updateGroups"]` exists and is a map:
   1. For each `key`, `value` in `response["updateGroups"]`:
      1. Let `owner` be equal to `key` parsed as an [ORIGIN], continuing the next
         iteration of this loop if there is an error.
      1. If `request context`'s `included_groups` does not contain `owner` as a
         key, continue the next iteration of this loop.
      1. If `value` is not a list, return failure.
      1. For each `element` in `value`:
         1. If `element` is not a map, continue the next iteration of this loop.
         1. If `element["index"]` does not exist or is not an integer or
            `element["updateIfOlderThanMs"]` does not exist or is not an integer, continue the
            next iteration of this loop.
         1. If `element["index"]` is not an integer or `element["index"] < 0`,
            continue the next iteration of this loop.
         1. If `element["index"]` is greater than or equal to the length of
            `included_groups[owner]`, continue the next iteration of this loop.
         1. Let `name` be the `interest group name` for `included_groups[owner][element]`.
         1. Let `interest group key` be the tuple (`owner`, `name`).
         1. Let `update duration` be `element["updateIfOlderThanMs"]`, parsed into a time
            duration as integer milliseconds.
         1. Set `processed response["update groups"][intereset group key]` to
            `update duration`.
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
1. If `response["buyerReportingId"]` exists and is a string, set
   `processed response["buyer reporting id"]` to `response["buyerReportingId"]`.
1. If `response["buyerAndSellerReportingId"]` exists and is a string, set
   `processed response["buyer and seller reporting id"]` to
   `response["buyerAndSellerReportingId"]`.
1. If `response["selectedBuyerAndSellerReportingId"]` exists and is a string, set
   `processed response["selected buyer and seller reporting id"]` to
   `response["selectedBuyerAndSellerReportingId"]`.
1. If `response["debugReports"]` exists and is an array:
   1. For each `per origin debug reports` in `response["debugReports"]`:
      1. If `per origin debug reports["adTechOrigin"]` does not exist or
         is not a string, continue with the next iteration.
      1. Let `ad tech origin` be `per origin debug reports[
         "adTechOrigin"]` parsed as an [ORIGIN], continue with the next
         iteration if there is an error.
      1. If `per origin debug reports["reports"]` does not exist or is
         not an array, continue with the next iteration.
      1. For each `report` in `per origin debug reports["reports"]`:
         1. If `report` is not a map, continue with the next iteration.
         1. Let `component win` be `report["componentWin"]` if it exists
            and is a bool, otherwise false.
         1. If `report["url"]` exists and is a string:
            1. Let `url` be `report["url"]` parsed as a [URL], or
               continue with the next iteration if there is an error.
            1. If `component win` is false, set
               `processed response["server filtered debugging only
               reports"][ad tech origin]` to `url`, and continue with
               the next iteration.
            1. Let `debug report key` be a new structure analogous to
               [server auction debug report key](https://wicg.github.io/turtledove/#server-auction-debug-report-key).
            1. Set `debug report key["from seller"]` to
               `report["isSellerReport"]` if it exists and is a bool,
               otherwise false.
            1. Set `debug report key["is debug win"]` to
               `report["isWinReport"]` if it exists and is a bool,
               otherwise false.
            1. Set `processed response[
               "component win debugging only reports"][
               debug report key]` to `url`.
         1. Otherwise:
            1. If `component win` is false and `processed response[
               "server filtered debugging only reports"]` does not
               contain `ad tech origin`, set `processed response[
               "server filtered debugging only reports"][
               ad tech origin]` to an empty list.
1. If `response["paggResponse"]` exists and is an array:
   1. For each `per origin response` in `pagg response`:
      1. If `per origin response` is not a map, continue with the next iteration.
      1. If `per origin response["reportingOrigin"]` does not exist or is not a string,
         continue with the next iteration.
      1. Let `reporting origin` be `per origin response["reportingOrigin"]` parsed as an [ORIGIN],
         continue with the next iteration if there is an error.
      1. If `per origin response["igContributions"]` does not exist or is not an array,
         continue with the next iteration.
      1. Let `names` be an empty array.
      1. If `request context`'s `included_groups` contains `owner` as a key, set `names` to its value.
      1. For each `ig contribution` in `per origin response["igContributions"]`:
         1. If `ig contribution` is not a map, continue with the next iteration.
         1. Let `coordinator` be null.
         1. If `ig contribution["coordinator"]` exists and is a string, set `coordinator` to
            `per origin response["reportingOrigin"]` parsed as an [ORIGIN], continue with the next
            iteration if there is an error.
         1. Otherwise if `ig contribution["igIndex"]` exists and is an integer:
            1. If `ig contribution["igIndex"] < 0` or is greater than or equal to the length of `names`,
                continue with the next iteration.
            1. Let `ig key` be the tuple (`owner`, `names[igIndex]`).
            1. If `request context`'s `ig pagg coordinators` contains `ig key`, set `coordinator` to
               `request context`'s `ig pagg coordinators[ig key]`.
          1. Let `is component win` be false.
          1. If `ig contribution["componentWin"]` exists and is a boolean, set `is component win` to it.
          1. If `ig contribution["eventContributions"]` exists and is an array:
             1. For each `event contribution` in `ig contribution["eventContributions"]`:
                1. Continue with the next iteration if any of the following conditions hold:
                   * `event contribution` is not a map;
                   * `event contribution["event"]` does not exist or is not a string;
                   * `event contribution["event"]` starts with "reserved.", but is not one of "reserved.win", "reserved.loss",
                     or "reserved.always", continue with the next iteration.
                1. Let `event` be `event contribution["event"]`.
                1. If `event contribution["contributions"]` exists and is an array, for each `contribution` in it:
                   1. Continue with the next iteration if any of the following conditions hold:
                      * `contribution` is not a map;
                      * `contribution["bucket"]` does not exist or is not a byte array or its size is greater than 16.
                      * `contribution["value"]` does not exist or is not an integer.
                   1. Let `private aggregation contribution` be a new structure analogous to [PAExtendedHistogramContribution]
                      (https://wicg.github.io/turtledove/#dictdef-paextendedhistogramcontribution).
                   1. Set `private aggregation contribution["bucket"]` to `contribution["bucket"]` parsed as a big endian integer.
                   1. Set `private aggregation contribution["value"]` to `contribution["value"]`.
                   1. If `is component win` is true:
                      1. Let `key` be a new structure analogous to [server auction private aggregation contribution key]
                         (https://wicg.github.io/turtledove/#server-auction-private-aggregation-contribution-key).
                      1. Set `key`["reporting origin"] to `reporting origin`.
                      1. Set `key`["coordinator"] to `coordinator`.
                      1. Set `key`["event"] to `event`.
                      1. If `processed response["component win private aggregation contributions"]` does not contain `key`, set
                         `processed response["component win private aggregation contributions"][key]` to a new array.
                      1. Append `private aggregation contribution` to `processed response["component win private aggregation contributions"][key]`.
                   1. Otherwise if `event contribution["event"]` starts with "reserved.", append `private aggregation contribution`
                      to `processed response["server filtered private aggregation contributions reserved"][key]`.
                   1. Otherwise, append `private aggregation contribution` to
                      `processed response["server filtered private aggregation contributions non reserved"][key]`.
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
