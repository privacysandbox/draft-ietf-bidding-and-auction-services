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

informative:
--- abstract

The Trusted Auction Service provides a way for advertising auctions to execute in a
remote trusted execution environment while preserving user privacy.

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
computation to take place on cloud servers in a trusted execution environment,
rather than running locally on a user's device. Moving computations to cloud in
a [Trusted Execution Environment (TEE)](https://github.com/privacysandbox/fledge-docs/blob/main/trusted_services_overview.md#trusted-execution-environment)
have the following benefits:

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
*   [trusted execution environment](https://github.com/privacysandbox/fledge-docs/blob/main/trusted_services_overview.md#trusted-execution-environment)
    can protect confidentiality of adtech code and signals.

Standardized protocols for interacting with Bidding and Auction Services are
essential to creating a diverse and healthy ecosystem for such services.

## Scope {#Scope}
This document provides a specification for the request-response message
architecture that is required to implement the Protected Audience API for executing
a remote, TEE-based ad auction.

This document does not describe distribution of private keys to trusted auction
services.

## Terminology          {#Terminology}

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL
NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED",
"MAY", and "OPTIONAL" in this document are to be interpreted as
described in BCP 14 {{!RFC2119}} {{!RFC8174}} when, and only when, they
appear in all capitals, as shown here.

# Message Format Specifications   {#format}

## Overview

To understand this document, it is important to know that the
communication between the browser and the remote servers uses a
request-response message exchange pattern. The request will first reach a seller server, after which
the seller will forward parts of the request to buyer servers. It is then up to the
seller server to gather buyer responses and form a final response for the browser. More detail
about the seller and buyer servers can be found in the [server-side system design documentation](https://github.com/privacysandbox/protected-auction-services-docs/blob/main/bidding_auction_services_system_design.md).

## Browser to Trusted Auction Server {#browser-to-server}

This section describes how the browser MUST form request messages in
order to communicate with the Trusted Auction Server.

### Request Payload Data

A request payload primarily consists of interest groups. An interest group
is represented by the following {{CDDL}}:

~~~~~ cddl
interestGroups = [ * interestGroup ]
interestGroup = {
  name: tstr,
  ? biddingSignalsKeys: [* tstr],
  ? userBiddingSignals: json,
  ? ads: [* adRenderId],
  ? components: [* adRenderId],
  ? browserSignals: {
    ? joinCount: int,
    ; Number of times the group was joined in the last 30
    ; days.

    ? bidCount: int,
    ; Number of times the group bid in an auction in the
    ; last 30 days.

    ? prevWins: [* [int, adRenderId]],
    ; Tuple of time-ad pairs for a
    ; previous win for this interest group that occurred
    ; in the last 30 days.
    ; The time is specified in seconds before the
    ; containing auctionBlob was requested.

    ? recencyMs: int
    ; The most recent join time for this group expressed
    ; in milli seconds before the containing auctionBlob
    ; was requested. This field will be used by newer client
    ; versions. For older devices, the precison will be in seconds.
    ; If recencyMs is present, this value will be used to offer
    ; higher precision. If not, recency will be used. Only
    ; one of the recency or recencyMs is expected to present in
    ; the request.
  }
}

adRenderId = tstr
json = tstr ; JSON encoded data
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
    * origin => bstr
    ; CBOR encoded list of interest groups compressed
    ; using the method described in `compression`.
  },
  ? enableDebugReporting: bool
}

origin = tstr .regexp "https://([^/:](:[0-9]+)?/"
uuid = tstr .regexp "[a-fA-F0-9]{8}-[a-fA-F0-9]{4}-[a-fA-F0-9]{4}-[a-fA-F0-9]{4}-[a-fA-F0-9]{12}"
~~~~~

The complete request MUST also be compressed using the same compression
algorithm as specified in the message framing ({{request-framing}}).

### Compression {#request-compression}

The payload MAY undergo compression.

The compression method's value in bits 4-0 in {{request-framing}}
corresponds to the below table:

| Compression | Description    |
| :---------: | :------------- |
|      0      | No Compression |
|      1      | Brotli         |
|      2      | GZIP           |
|    3-31     | Reserved       |


### Framing and Padding {#request-framing}

The plaintext message has the following framing:

| Byte     | 0         | 0             | 1 to 4   | 5 to Size+4       | Size+5 to end   |
| -------- | --------- | ------------- | -------- | ----------------- | --------------- |
| Bits     | 7-5       | 4-0           | *        | *                 | *               |
| -------- | --------- | ------------- | -------- | ----------------- | --------------- |
| Contents | Version   | Compression   | Size     | Request Payload   | Padding         |

where the the first 3 bits of the frame header specify the payload version and
the following 5 bits specify the compression. The format described in this
document corresponds to version 0.

Messages SHOULD be zero padded up to one of the following bin sizes, where kB is
understood to mean 1024 bytes: 0kB, 5kB, 10kB, 20kB, 30kB, 40kB, 55kB. Messages
SHALL NOT be be larger than 55kB. An implementation may need to remove some
data from the payload to fit inside the largest bucket.

### Encryption

After framing and padding the compressed payload, the entire plaintext message is
encrypted using HPKE with the encapsulation performed according
to {{OHTTP}}. Since we are repurposing the OHTTP encapsulation mechanism, we
define the media type "message/auction request" which is used for encryption.


## Trusted Auction Server To Browser {#server-to-browser}

This section describes how the browser MUST interpret response messages from
the Trusted Auction Server.

### Decryption

The response message is encrypted using HPKE with the encapsulation performed
according to {{OHTTP}} as the response to the request message. Since we are
repurposing the OHTTP encapsulation mechanism, we define the media type
"message/auction response" which is used for encryption. The browser SHALL
decrypt the response by following the standard {{OHTTP}} [Encapsulated Response
decryption procedure](https://www.rfc-editor.org/rfc/rfc9458#section-4.4-5).

### Decompression

The message framing is as in {{request-framing}}, but the entire response payload is
compressed. The Server shall zero pad the response (TODO).

### Response Payload Data

The response has the following data, serialized as {{CBOR}} and matching
the following shape (described via {{CDDL}}):

~~~~~cddl
response = {
  adRenderURL: tstr,
  ; The ad that will be rendered

  ? components: [* tstr],
  ; List of render URLs for
  ; component ads to be displayed
  ; as part of this ad

  ? interestGroupName: tstr,
  ; Name of the InterestGroup to which
  ; the ad belongs

  ? interestGroupOwner: origin,
  ; Origin of the Buyer who owns
  ; the interest group

  ? biddingGroups: {
    * origin => [* int]
    ; Indices of interest groups
    ; in the original request for
    ; this owner that submitted a bid
  },

  ? score: float,
  ; Score of the ad determined
  ; during the auction.
  ; Any value that is zero or negative indicates
  ; that the ad cannot win the auction.
  ; The winner of the auction would be the
  ; ad that was given the highest score.

  ? bid: float,
  ; Bid price corresponding to an ad

  ? bidCurrency: tstr .size 3 .regexp /^[A-Z]{3}$/,
  ; Optional currency of the bid.
  ; Three uppercase letters, ISO 4217 alpha code suggested.

  ? buyerReportingId: tstr,
  ; Optional BuyerReportingId of
  ; the winning Ad

  ? buyerAndSellerReportingId: tstr,
  ; Optional
  ; BuyerAndSellerReportingId of
  ; the winning Ad

  ? isChaff: bool,
  ; The auction result
  ; may be ignored if set to true.

  ? winReportingUrls: {
    ? buyerReportingUrls: reportingUrls,
    ? componentSellerReportingUrls: reportingUrls,
    ? topLevelSellerReportingUrls: reportingUrls
  },

  ? error: {
    code: int,
    message: tstr
  },

  ? adMetadata: json,
  ; Arbitrary metadata to pass
  ; to the top-level seller

  ? topLevelSeller: origin,
  ; Optional name/domain for
  ; top-level seller in case this
  ; is a component auction

  ? kAnonWinnerJoinCandidates: [* KAnonJoinCandidate],

  ? kAnonWinnerPositionalIndex: int,
  ; Positional index of the
  ; k-anon winner.
  ; Note: Positional index \u003e= 0.
  ; If this is equal to 0, that would imply the
  ; highest scored bid is also K-Anonymous and hence a winner.
  ; If this is greater than 0,
  ; the positional index would imply the index of
  ; first K-Anonymous scored bid in a sorted
  ; list in decreasing order of scored bids.
  ; In this case, the highest scored bid that
  ; is not K-Anonymous is the ghost winner.
  ; In case all scored bids fail the K-Anonymity constraint,
  ; this would be set to -1 since there is no winner.
  ; In case all scored bids \u003c= 0,
  ; this would be set to -1 since there is no winner.

  ? kAnonGhostWinners: [* KAnonGhostWinner]
}

origin = tstr .regexp "https://([^/:](:[0-9]+)?/"
json = tstr ; JSON encoded data

; Defines the structure for reporting URLs
reportingUrls = {
  ? reportingUrl: tstr,
  ? interactionReportingUrls: { * tstr => tstr }
}

; Join candidates for K-Anonymity
KAnonJoinCandidate = {
  adRenderUrlHash: tstr,
  ; SHA-256 hash
  ; of the tuple: render_url, interest group
  ; owner, reportWin() UDF endpoint.

  ? adComponentRenderUrlsHash: [* tstr],
  ; SHA-256 hash
  ; of an ad_component_render_url
  ; Note: There is a maximum limit of 40 ad
  ; component render urls per render url.

  reportingIdHash: tstr
  ; Protected Audience: SHA-256 hash
  ; should include IG owner, ad render url,
  ; reportWin() UDF url and one of the
  ; following:
  ; - If buyerAndSellerReportingId exists
  ; - Else if buyerReportingId exists
  ; - Else IG name should be included
  ; Note: An adtech can use either
  ; buyerReportingId or
  ; buyerAndSellerReportingId
}

; Data for the ghost winner sent back to the client
; This should also include key-hashes corresponding to the
; ghost winning ad
; Refer https://wicg.github.io/turtledove/#k-anonymity
KAnonGhostWinner = {
  kAnonJoinCandidates: [* KAnonJoinCandidate],
  ; Join candidates for the
  ; K-Anon ghost winner

  ? interestGroupIndex: int,
  ; Interest group index
  ; corresponding to buyer
  ; that generated the ghost
  ; winning bid

  owner: origin,
  ; Origin (Chrome) of the buyer
  ; who owns the ghost winner

  ? ghostWinnerPrivateAggregationSignals: {
    bucket: bytes,
    ; 128 bit integer in the form
    ; of bytestring

    value: int
  },
  ; Private aggregation signals for the ghost winner
  ; Single seller auctions: This would correspond to a ghost
  ; winner if available
  ; Note: Event type is "reserved.loss" and bid
  ; rejection reason is 8 when K-Anonymity threshold is not met.

  ? ghostWinnerForTopLevelAuction: {
    adRenderUrl: tstr,
    ; The ad render url corresponding
    ; to ghost winner

    ? adComponentRenderUrls: [* tstr],
    ; Render URLs for ads which are
    ; components of the main ghost winning ad

    modifiedBid: float,
    ; Modified bid price corresponding
    ; to ghost winning bid

    ? bidCurrency: tstr,
    ; Optional. Indicates the currency
    ; used for the bid price

    ? adMetadata: json,
    ; Arbitrary metadata associated
    ; with ghost winner to pass to the
    ; top-level seller

    buyerAndSellerReportingId: tstr
    ; BuyerAndSellerReportingId of
    ; the ghost winning Ad
  }
  ; In case of multiseller auction, the associated data for the
  ; ghost winner will be returned so that the ghost winning bid can
  ; be scored during the top level auction.
}
~~~~~

# Security Considerations

TODO

# IANA Considerations

TODO

--- back

# Acknowledgments
{:numbered="false"}

TODO
