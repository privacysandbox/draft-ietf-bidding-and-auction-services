---
coding: utf-8

title: Bidding and Auction Services
docname: draft-ietf-bidding-and-auction-services-latest
category: std

# area: TODO
# workgroup: TODO
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
format that a browser can use to communicate with remote services
that allows the browser to offload much of the work involved in running an advertisement
selection auction as part of the browser's implementation of the
Protected Audience API.

This document does not describe distribution of private keys to the bidding
and auction services.

## Terminology {#Terminology}

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL
NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED",
"MAY", and "OPTIONAL" in this document are to be interpreted as
described in BCP 14 {{!RFC2119}} {{!RFC8174}} when, and only when, they
appear in all capitals, as shown here.

# Message Format Specifications {#format}

## Overview

To understand this document, it is important to know that the
communication between the browser and the remote servers uses a
request-response message exchange pattern. The request will first reach a seller server, after which
the seller will forward parts of the request to buyer servers. It is then up to the
seller server to gather buyer responses and form a final response for the browser. More detail
about the seller and buyer servers can be found in the [server-side system design documentation](https://github.com/privacysandbox/protected-auction-services-docs/blob/main/bidding_auction_services_system_design.md).

### Common Definitions {#common-definitions}

{{format}} makes frequent use of the following definitions.

| Term with CDDL Definition | Detailed Reference |
| :--- | :--- |
| `json = tstr` | [JSON] |
| `uuid = tstr .regexp "[a-fA-F0-9]{8}-[a-fA-F0-9]{4}-[a-fA-F0-9]{4}-[a-fA-F0-9]{4}-[a-fA-F0-9]{12}"` | {{!RFC9562}} |
| `origin = tstr .regexp "https://([^/:](:[0-9]+)?/"` | [ORIGIN] |
| `currency = tstr .size 3 .regexp /^[A-Z]{3}$/` | [ISO4217] |
| `adRenderUrl = tstr` | [URL] |
| `adRenderId = tstr` | [ADRENDERID] |
| `interestGroupOwner = origin` | TODO |

## Browser to Bidding and Auction Services {#browser-to-services}

This section describes how the browser MUST form and serialize request messages
in order to communicate with the Bidding and Auction services.

The required inputs from the browser to the steps in {{browser-to-services}}
are:

1. A complete request payload, as defined in {{complete-request}}.
2. A map of `interestGroupOwner` to interest group names, as defined in {{ig-map}}.
3. An encryption [Key Configuration](https://www.rfc-editor.org/rfc/rfc9458#name-key-configuration).
See {{encryption}} for more detail on this requirement.

The outputs of the steps in {{browser-to-services}} will result in:

1. The context blob ({{hpke-context}}) used to encrypt the response and to be later used in {{decryption}}.
2. An encrypted byte string ({{encapsulation}}) representing the request to be sent to the Bidding
and Auction Services.

### Request Payload Data {#request-payload}

A request payload primarily consists of interest groups. The interest groups are
then wrapped into a request object that contains additional data.

#### Interest Groups {#interest-groups}

A list of interest group is represented by the following [CDDL]:

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

#### Complete Request {#complete-request}

A list of interest groups for each owner MUST first be represented as {{CBOR}},
and then the serialized list MUST be indiviudally compressed according to the
compression algorithm specified in the message framing ({{request-framing}}).
This compressed interest group data MUST then be aggregated into a map in the
complete request, which MUST be {{CBOR}} represented by the following {{CDDL}}:

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

The complete request MUST also be compressed using the same compression
algorithm as specified in the message framing ({{request-framing}}).

### Compression {#request-compression}

The payload MAY undergo compression.

The compression method's value in bits 4-0 in {{request-framing}}
corresponds to the below table:

| Compression | Description        |
| :---------: | :-------------     |
|      0      | No Compression     |
|      1      | Brotli {{!RFC7932}}|
|      2      | GZIP {{!RFC1952}}  |
|    3-31     | Reserved           |


### Framing and Padding {#request-framing}

The plaintext message has the following framing:

| Byte     | 0         | 0             | 1 to 4   | 5 to Size+4       | Size+5 to end   |
| -------- | --------- | ------------- | -------- | ----------------- | --------------- |
| Bits     | 7-5       | 4-0           | *        | *                 | *               |
| -------- | --------- | ------------- | -------- | ----------------- | --------------- |
| Contents | Version   | Compression   | Size     | Request Payload   | Padding         |

where the the first 3 bits of the frame header specify the payload
version and the following 5 bits specify the compression algorithm.
The format described in this document corresponds to version 0.

Messages MAY be zero padded so that the encrypted request is one of the
following bin sizes: 0KiB, 5KiB, 10KiB, 20KiB, 30KiB, 40KiB, 55KiB. An
implementation MAY need to remove some data from the payload to fit inside the
largest bucket.

A compatible implementation processing requests SHOULD NOT rely on a specific
padding scheme for requests.

### Encryption {#encryption}

After framing and padding the compressed payload, the entire plaintext message is
encrypted using [HPKE] with the encapsulation performed similarly
to [Section 4.3](https://www.rfc-editor.org/rfc/rfc9458#section-4.3) of {{OHTTP}}.
Details on how to acquire [HPKE] keys are out of scope for this document, however
we assume they will be provided by the browser as a [Key Configuration](https://www.rfc-editor.org/rfc/rfc9458#name-key-configuration)
and are required inputs to the context formation ({{context}}) and encapsulation
({{encapsulation}}) processes.

#### Encryption Context {#context}

The encryption context MUST be composed of two parts:

1. HPKE Context ({{hpke-context}})
2. A map storing interest group information ({{ig-map}})

The two parts may be stored in whichever way is most convenient, as long as
they are provided to the browser as a single opaque blob.

##### HPKE Context {#hpke-context}

[Section 5.1](https://www.rfc-editor.org/rfc/rfc9180.html#name-creating-the-encryption-con)
of [HPKE] describes how to create the Context required for encryption. This is
considered the 'sending' HPKE context, and the browser MUST store it for later
use when decrypting the response, as described in {{decryption}}.

##### Interest Group Map {#ig-map}

The encryption context must include a map of `interestGroupOwner` ({{common-definitions}})
to a list of plaintext string interest group names (the order of which MUST
exactly match the request payload provided in {{interest-groups}}). This map
will be used by {{response-payload}} to deserialize the response into a
a structure usable by the browser.

This map may be represented in whichever way is most convenient for use in
{{response-payload}}.

#### Encapsulation {#encapsulation}
Instead of encapsulating Binary HTTP [BINARY] as per [Step 1 in OHTTP](https://www.rfc-editor.org/rfc/rfc9458#section-4.3-4.1.1),
the output from {{request-framing}} MUST be used as-is. This means that
we are repurposing the [OHTTP] encapsulation mechanism, so [we are required to
define new media types](https://www.rfc-editor.org/rfc/rfc9458.html#name-repurposing-the-encapsulati):

* The OHTTP request media type is “message/auction request”
* The OHTTP response media type is “message/auction response”

Note that these media types are [concatenated with other fields when
creating the HPKE encryption context](https://www.rfc-editor.org/rfc/rfc9458.html#name-encapsulation-of-requests),
and are not HTTP content or media types. In order to perform the encapsulation,
the steps in [the OHTTP encapsulation process](https://www.rfc-editor.org/rfc/rfc9458#section-4.3-3)
MUST be followed precisely, except with an `info` equivalent to `message/auction request`.

### Payload Optimization {#request-optimization}

A compatible implementation MAY support control over how interest groups are
serialized into a request and the size of the request. An example implementation
for this feature is as follows:

This algorithm takes as input all of the `relevant interest groups`, an optional
`desired total size`, and an optional list of `interest group owners` to include
each with an optional `desired size`.

1. If `desired total size` is not specified, but the list of `interest group owners`
   includes at least one entry with a specified `desired size`:
   1. Set `desired total size` to the sum of all specified `desired size` in the
      list of `interest group owners`.
1. Group the list of `relevant interest groups` by owner into a map of from
   interest group owner to a list of interest groups sorted by decreasing
   priority, `interest group map`.
1. If the list of `interest group owners` is specified, remove interest groups
   whose owner is not on the list.
1. Construct a request, `request`, as normal only with the value of the
   `interestGroups` field for each interest group owner as
   zero length.
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
   1. Serialize the `interest group list` into `serialized_group`.
   1. If adding the `serialized_group` to `request` would make it more than
      `allowed_interest_group_size` larger than the current size, then remove
      the lowest priority interest group and repeat from the previous step.
   1. Set `request["interestGroups"][interest group owner]` to
      `serialized_group`.
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
   1. Serialize the `interest group list` into `serialized_group`.
   1. If adding the `serialized_group` to `request` would make it more than
      `allowed_interest_group_size` larger than the current size, then remove
      the lowest priority interest group and repeat from the previous step.
   1. Set `request["interestGroups"][interest group owner]` to
      `serialized_group`.
   1. Set `current_size` to be the serialized size of the encrypted request
      created from `request` without padding.
1. If there are no interest groups in the request, discard the `request` and
   return an empty byte array.
1. Frame `request` as in {{request-framing}} and zero pad up to `desired total size`.
1. Return the encrypted result (as in {{encryption}}).

## Bidding and Auction Services to Browser {#services-to-browser}

This section describes how the browser MUST deserialize response messages from
the Bidding and Auction Services. The steps MUST be performed in the following
order: decryption ({{decryption}}), decompression ({{decompression}}), and
finally parsing the response payload as in {{response-payload}}.

The required inputs from the browser to the steps in {#services-to-browser}:

1. The encrypted response byte string.
2. The HPKE context created in {{hpke-context}}.

The resulting output for the browser is:

1. The decoded response data structure, as described in {{response-payload}}.

### Decryption {#decryption}

The response message is encrypted using HPKE with the encapsulation performed
according to [Section 4.4](https://www.rfc-editor.org/rfc/rfc9458.html#name-encapsulation-of-responses)
of {{OHTTP}} as the response to the request message.

The browser MUST decrypt the response by following the standard {{OHTTP}} [Encapsulated Response
decryption procedure](https://www.rfc-editor.org/rfc/rfc9458#section-4.4-5). The
context used for decryption MUST be precisely the same context blob that was
created in {{hpke-context}}, and should be provided by the browser in addition to
the response byte string.

### Decompression {#decompression}

The message framing is exactly as in {{request-framing}}, but the entire
response payload is compressed. Starting with Byte 0, read bits 4-0 and use
the chart in {{request-compression}} to decode which decompression algorithm
MUST be applied to the response payload.

The output of the decompression process will be a payload equivalent to the
[auction server response struct](https://wicg.github.io/turtledove/#server-auction-response),
as further described in {{response-payload}}.

### Response Payload Data {#response-payload}

After decompression, the result will be a {{CBOR}} byte string (described below).

The byte string MUST be decoded into a format usable by the browser,
the [server auction response structure](https://wicg.github.io/turtledove/#server-auction-response).
The `server auction response structure` is the final, consumable output of {{services-to-browser}}.

To perform this decoding, use a compliant [CBOR] decoder. The following [CDDL]
describes the structure of the CBOR data. Map the decoded data
to the `server auction response structure` according to the instructions
provided for each field.

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
  ; Else, TODO
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
  ; Maps to https://wicg.github.io/turtledove/#server-auction-reporting-info-reporting-url.
  ; If not present, map as Null.
  ? reportingUrl: tstr,
  ; Maps to https://wicg.github.io/turtledove/#server-auction-reporting-info-beacon-urls.
  ; If not present, map as an empty ordered map (https://infra.spec.whatwg.org/#ordered-map).
  ; Else, TODO
  ? interactionReportingUrls: { * tstr => tstr }
}

~~~~~

# Security Considerations

TODO

# IANA Considerations

This document introduces no additional considerations for IANA.

--- back

# Acknowledgments
{:numbered="false"}

TODO
