---
coding: utf-8

title: Trusted Auction Service
docname: draft-ietf-trusted-auction-service-latest
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
  OHTTP: RFC9458
  SHA-256: RFC6234
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

The Trusted Auction Service provides a way for advertising auctions to execute in a
remote environment while preserving user privacy.

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

This Trusted Auction Service proposal outlines a way to allow Protected Audience
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
format that a browser can use to communicate with trusted remote services
that allows the browser to offload much of the work involved in running an advertisement
selection auction as part of the browser's implementation of the
Protected Audience API.

This document does not describe distribution of private keys to trusted auction
services.

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

### Common Definitions

{{format}} makes frequent use of the following definitions.

| Term with CDDL Definition | Detailed Reference |
| :--- | :--- |
| `json = tstr` | TODO |
| `uuid = tstr .regexp "[a-fA-F0-9]{8}-[a-fA-F0-9]{4}-[a-fA-F0-9]{4}-[a-fA-F0-9]{4}-[a-fA-F0-9]{12}"` | TODO |
| `origin = tstr .regexp "https://([^/:](:[0-9]+)?/"` | [ORIGIN] |
| `currency = tstr .size 3 .regexp /^[A-Z]{3}$/` | TODO |
| `adRenderUrl = tstr` | [URL] |
| `adRenderId = tstr` | [ADRENDERID] |
| `interestGroupName = tstr` | TODO |
| `interestGroupOwner = origin` | TODO |

## Browser to Trusted Auction Server {#browser-to-server}

This section describes how the browser MUST form request messages in
order to communicate with the Trusted Auction Server.

### Request Payload Data

A request payload primarily consists of interest groups. An interest
group is represented by the following {{!CDDL}}:

~~~~~ cddl
interestGroups = [ * interestGroup ]
interestGroup = {
  ; This interest group's name, see
  ; https://wicg.github.io/turtledove/#interest-group-name.
  name: interestGroupName,

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

Each interest group MUST first be represented as {{CBOR}}, and then MUST
be indiviudally compressed according to the compression algorithm
specified in the message framing ({{request-framing}}). This compressed
interest group data MUST then be aggregated into a map in the complete
request, which MUST be {{CBOR}} represented by the following {{CDDL}}:

~~~~~ cddl
request = {
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

Messages SHOULD be zero padded up to one of the following bin sizes:
0KiB, 5KiB, 10KiB, 20KiB, 30KiB, 40KiB, 55KiB. Messages
SHALL NOT be be larger than 55KiB. An implementation may need to remove some
data from the payload to fit inside the largest bucket.

### Encryption {#encryption}

After framing and padding the compressed payload, the entire plaintext message is
encrypted using HPKE with the encapsulation performed according
to {{OHTTP}}.

Since we are [repurposing the OHTTP encapsulation mechanism, we are
required to define new media types](https://www.rfc-editor.org/rfc/rfc9458.html#name-repurposing-the-encapsulati):

* The OHTTP request media type is “message/auction request”
* The OHTTP response media type is “message/auction response”

Note that these media types are [concatenated with other fields when
creating the HPKE encryption context](https://www.rfc-editor.org/rfc/rfc9458.html#name-encapsulation-of-requests),
and are not HTTP content or media types.

## Auction Server

TODO

## Trusted Auction Server To Browser {#server-to-browser}

This section describes how the browser MUST interpret response messages from
the Trusted Auction Server.

### Decryption

The response message is encrypted using HPKE with the encapsulation performed
according to {{OHTTP}} as the response to the request message. See {{encryption}} for more details.
The browser MUST decrypt the response by following the standard {{OHTTP}} [Encapsulated Response
decryption procedure](https://www.rfc-editor.org/rfc/rfc9458#section-4.4-5).

### Decompression

The message framing is as in {{request-framing}}, but the entire response payload is
compressed. The Server shall zero pad the response (TODO).

### Response Payload Data

The response has the following data, serialized as {{CBOR}} and matching
the following shape (described via {{CDDL}}):

~~~~~cddl
response = {
  ; The ad to render.
  adRenderURL: adRenderUrl,

  ; List of URLs for component ads displayed as part of this
  ; ad.
  ? components: [* adRenderUrl],

  ; Name of the interest group to which the ad belongs.
  ? interestGroupName: interestGroupName,

  ; Origin of the Buyer who owns the interest group.
  ; The original request for this response MUST contain this
  ; interestGroupOwner, which additionally MUST provide an interest
  ; group with interestGroupName.
  ? interestGroupOwner: interestGroupOwner,

  ; Indices of interest groups in the original request for this owner
  ; that submitted a bid.
  ? biddingGroups: {
    * interestGroupOwner => [* int]
  },

  ; Score of the ad determined during the auction.
  ; Any value that is zero or negative indicates that the ad cannot
  ; win the auction.
  ; The winner of the auction would be the ad that was given the
  ; highest score.
  ? score: float,

  ; Bid price corresponding to an ad
  ? bid: float,

  ; Optional currency of the bid.
  ? bidCurrency: currency,

  ; Optional BuyerReportingId of the winning Ad
  ? buyerReportingId: tstr,

  ; Optional BuyerAndSellerReportingId of the winning Ad
  ? buyerAndSellerReportingId: tstr,

  ; The auction result may be ignored if set to true.
  ? isChaff: bool,

  ? winReportingUrls: {
    ? buyerReportingUrls: reportingUrls,
    ? componentSellerReportingUrls: reportingUrls,
    ? topLevelSellerReportingUrls: reportingUrls
  },

  ? error: {
    code: int,
    message: tstr
  },

  ; Arbitrary metadata to pass to the top-level seller
  ? adMetadata: json,

  ; Optional name/domain for the top-level seller in case this is a
  ; component auction.
  ? topLevelSeller: origin,

  ? kAnonWinnerJoinCandidates: [* KAnonJoinCandidate],

  ; Positional index of the k-anon winner.
  ; Note: Positional index >= 0.
  ; If this is equal to 0, the highest scored bid is also K-Anonymous
  ; and hence a winner.
  ; If this is greater than 0, the positional index implies the index
  ; of the first K-Anonymous scored bid in a sorted list in
  ; decreasing order of scored bids.
  ; In this case, the highest scored bid that is not K-Anonymous is
  ; the ghost winner.
  ; In case all scored bids fail the K-Anonymity constraint, this
  ; would be set to -1 since there is no winner.
  ; In case all scored bids <= 0, this would be set to -1 since
  ; there is no winner.
  ? kAnonWinnerPositionalIndex: int,

  ? kAnonGhostWinners: [* KAnonGhostWinner]
}

; Defines the structure for reporting URLs.
reportingUrls = {
  ? reportingUrl: tstr,
  ? interactionReportingUrls: { * tstr => tstr }
}

; Join candidates for K-Anonymity
KAnonJoinCandidate = {
  ; SHA-256 [RFC6234] hash.
  ; Must be computed according to:
  ; https://wicg.github.io/turtledove/#compute-the-key-hash-of-ad
  adRenderUrlHash: tstr,

  ; SHA-256 [RFC6234] hash.
  ; MUST be computed according to:
  ; https://wicg.github.io/turtledove/#compute-the-key-hash-of-component-ad
  ; Note: There is a maximum limit of 40 ad component render urls per
  ; render url.
  ? adComponentRenderUrlsHash: [* tstr],

  ; SHA-256 [RFC6234] hash.
  ; MUST be computed according to:
  ; https://wicg.github.io/turtledove/#compute-the-key-hash-of-reporting-id
  reportingIdHash: tstr
}

; Data for the ghost winner sent back to the client.
; This should also include key-hashes corresponding to the ghost
; winning ad.
; Refer https://wicg.github.io/turtledove/#k-anonymity
KAnonGhostWinner = {
  ; Join candidates for the K-Anon ghost winner.
  kAnonJoinCandidates: [* KAnonJoinCandidate],

  ; Interest group index of the buyer who generated the ghost winning
  ; bid.
  ? interestGroupIndex: int,

  ; Buyer index who generated the ghost winning bid.
  ? buyerIndex: int,

  ; Origin of the buyer who owns the ghost winner.
  owner: interestGroupOwner,

  ; Private aggregation signals for the ghost winner.
  ; In single seller auctions, this represents a ghost winner if
  ; available.
  ; Note: Event type is "reserved.loss" and bid rejection reason is 8
  ; when the K-Anonymity threshold is not met.
  ? ghostWinnerPrivateAggregationSignals: {
    ; 128-bit integer in bytestring format.
    bucket: bytes,

    value: int
  },

  ; In multi-seller auctions, this data allows scoring the ghost
  ; winning bid during the top-level auction.
  ? ghostWinnerForTopLevelAuction: {
    ; Ad render URL of the ghost winner.
    adRenderUrl: adRenderUrl,

    ; Render URLs for component ads of the main ghost winning ad.
    ? adComponentRenderUrls: [* tstr],

    ; Modified bid price of the ghost winning bid.
    modifiedBid: float,

    ; Optional currency used for the bid price.
    ? bidCurrency: currency,

    ; Arbitrary metadata associated with the ghost winner, passed to
    ; the top-level seller.
    ? adMetadata: json,

    ; BuyerAndSellerReportingId of the ghost winning ad.
    buyerAndSellerReportingId: tstr
  }
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