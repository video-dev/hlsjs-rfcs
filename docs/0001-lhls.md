- Feature Name: Low-Latency Streaming
- Start Date: 2018-08-18
- RFC PR: https://github.com/video-dev/hlsjs-rfcs/pull/1
- Hls.js Issue: (leave this empty)
- Authors: John Bartos, Tom Boshoven
- Signatories: JW Player, (your name here!)

# Summary
[summary]: #summary

Low-latency streaming is becoming an increasingly desired feature for live events, and is typically defined as a delay of two seconds or less from point of capture to playback (glass-to-glass). However, the current HLS specification precludes this possibility - within the HLS guidelines, the best attempts have achieved about four seconds glass-to-glass, with average implementations typically beyond thirty seconds. This RFC proposes modifications to the HLS specification ("HTTP Live Streaming 2nd Edition" specification (IETF RFC 8216, draft 03)") which aim to reduce the glass-to-glass latency of a live HLS stream to two seconds or below. The scope of these changes are centered around a new "prefetch" segment; it's advertising, delivery, and interpretation within the client.

# Goals and Motivation
[goals-and-motivation]: #goals-and-motivation

Our goal is to reduce latency to about two seconds; to build LHLS into a Hls.js, a free, open, and popular client; and to foster an ecosystem of low-latency solutions through an open standard. The meta-purpose of this proposal is to gain consensus on an implementation before Hls.js builds it. We want consensus from developers interested in using Hls.js as their LHLS client - both backend and frontend developers. Consensus is sought because Hls.js cannot afford to build a bespoke LHLS implementation for each proprietary scheme or desired use case. That approach does not scale, nor is it appropriate for an open-source project to meet the needs of an individual. Through the peer review of this proposal we hope to arrive at a single, technically sound solution which addresses the varied requirements of our developer community - to build something that developers can and will use. It is also our hope that other HLS clients will implement this proposal and help drive its adoption.

Proposed LHLS use cases include (but are not restricted to) time-sensitive live events and live interactivity. Some examples of these are an enhanced live sports experience (a solution to the "cheering problem"), companion screens, and audience-driven live gaming.

### Principles

1. Accessible

An accessible solution is one which will actually be implemented. This proposal strives to be easily implementable, both client and server-side, by leveraging as much existing technology as possible. Any new code is kept to a minimum and follows established HLS patterns.

2. Scalable

A scalable solution must be able to deliver low-latency video at the existing scale of HLS. Scale here does not just mean viewer numbers, but also the breadth of use cases. The client must be empowered to play low-latency streams while supporting as many uses of HLS as possible (e.g. SSAI); and the server must be able to deliver an LHLS in a cost-effective manner.

3. Good Enough

A pragmatic solution is as good as it needs to be, but balances performance with the above considerations. A solution which achieves two seconds of latency with broad support is twice as good as a solution which achieves 1 second with few adopters.

# Guide-Level Explanation
[guide-level-explanation]: #guide-level-explanation

Low-latency streaming is used by Hls.js to stream HLS segments which have not completed transcoding yet. Segments do not need to be complete to be buffered - each segment contains a number of GOPs (group of pictures), and each GOP can be independently transmuxed and appended to a Media Source Extensions (MSE) source buffer in the playback user agent. Using HTTP chunked transfer encoding, Hls.js is able to maintain a persistent connection with a transcoding server; the server, in turn, is able to push partial segments to the client before completion. Additional reductions in latency are possible by negotiating the TCP connection ahead of time the segment needs to be buffered.

Hls.js takes advantage of this feature when it detections a custom Media Segment tag within a playlist. When Hls.js sees this tag, it initiates a connection to the segment endpoint using the Fetch API with streaming body responses. Each time Hls.js receives a chunk of data, it passes it through it's transmuxing pipeline so that it may be buffered. Any samples which cannot be decoded using the information in the present chunk are saved so that they may be used in the next. In this way Hls.js is able to stream segments through it's pipeline instead of waiting for the whole segment to be available first.

The structure of a low-latency playlist ensures backwards compatibility not only with older versions of Hls.js, but also with clients which do not support this proposal. Low-latency segments are advertised with custom tags which, according to the HLS spec, must be ignored by clients. And in addition to low-latency segments an LHLS playlist also advertises complete segments. It is this feature which allows servers to deliver a single manifest which supported by all clients; and provides supporting clients the option to disable low-latency streaming at-will.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This section details how Hls.js interprets a low-latency playlist. Developers wishing to use LHLS must adhere to the conventions laid out within this section. "The client" here refers to Hls.js but is worded so that any MSE-based player may fulfill the same requirements.

## Compliance with the HLS Specification

This specification is a superset of the "HTTP Live Streaming 2nd Edition" specification (IETF RFC 8216, draft 03)". Unless otherwise noted, all language contained within applies here. For example, if there is no mention of modification to a item (e.g. `EXT-X-PROGRAM-DATE-TIME`), then all language in RFC8261 pertaining to that item applies.

This specification provides no additional constraint on media type used. As of writing this, RFC8216 draft 03 supports MPEG-TS and fragmented MP4, including CMAF fmp4. Support for these formats is dependent on the client.

## Glossary

- Client: An MSE-Based JS player such as Hls.js
- Server: The system comprising delivery, transcoding, cdn etc.; the party responsible for creating and serving the manifest
- Prefetch segment: A segment which has been advertised but is not yet available
- Complete segment: A segment which has been advertised and available. Any non-LHLS playlist consists only of complete segments.

## Prefetch Media Segments

The `EXT-X-PREFETCH` tag specifies a prefetch segment. The server may advertise zero or more prefetch segments within a Media Playlist. These segments must appear after all complete segments. Its format is:

`#EXT-X-PREFETCH: <URI>`

To each prefetch segment response, the server must append the `Transfer-Encoding: chunked` header. The server must maintain the persistent HTTP connection long enough for a client to receive the entire segment - this must be no less than the time from when the segment was first advertised to the time it takes to complete.

Any constraint in the HLS spec applying to complete segments also applies to prefetch segments unless otherwise specified. For example:
- A segment must remain available to clients for a period of time equal to the duration of the segment plus the duration of the longest Playlist file distributed by the server containing that segment.

The client may choose to ignore prefetch segments.

### Sequence Numbers

A prefetch segment's Discontinuity Sequence Number is the value of the `EXT-X-DISCONTINUITY-SEQUENCE` tag (or zero if none) plus the number of `EXT-X-DISCONTINUITY` and `EXT-X-PREFETCH-DISCONTINUITY` tags in the Playlist preceding the URI line of the segment.

If a prefetch segment is the first segment in a manifest, its Media Sequence Number is either 0, or declared in the Playlist. The Media Sequence Number of every other prefetch segment is equal to the Media Sequence Number of the complete segment or prefetch segment that precedes it plus one.

### Playlist Duration

The duration of a Media Playlist containing prefetch segments is considered to be equal to the duration of all complete segments plus the expected duration of all prefetch segments.

## Media Segment Tags

The server must not precede any prefetch segment with metadata other than those specified in this document, with the specified constraints.

### EXTINF

A prefetch segment must not be advertised with an `EXTINF` tag. The duration of a prefetch segment must be equal to or less than what is specified by the `EXT-X-TARGETDURATION` tag. 

### EXT-X-DISCONTINUITY

A prefetch segment must not be advertised with an `EXT-X-DISCONTINUITY` tag. To insert a discontinuity just for prefetch segments, the server must insert the `EXT-X-PREFETCH-DISCONTINUITY` tag before the newest `EXT-X-PREFETCH` tag of the new discontinuous range. 

### EXT-X-MAP

Prefetch segments must not be advertised with an `EXT-X-MAP` tag.

### EXT-X-KEY

Prefetch segments may be advertised with an `EXT-X-KEY` tag. The key itself must be complete; the server must not expect the client to progressively stream keys.

## Transformation to Complete Segments

After the server has made a prefetch segment available for direct playback, that segment must be transformed to a complete segment. The following transform constraints must be observed:

- The server must not insert any segment between oldest completed segment and the newest prefetch segment.
- The server must advertise the transformed segment with the same URI used in the prefetch advertisement.
- The server must append the `EXTINF` metadata tag to the transformed segment.
- If the prefetch segment was advertised with a metadata tag, the transformed segment must be advertised with an identical or equivalent metadata tag. For example:
    - `EXT-X-PREFETCH-DISCONTINUITY` must be transformed to `EXT-X-DISCONTINUITY`
- When a prefetch segment is transformed to a complete segment, the server must not increment the `EXT-X-MEDIA-SEQUENCE` value. 
- When a prefetch discontinuity is transformed to complete discontinuity, the server must not increment the `EXT-X-DISCONTINUITY-SEQUENCE` value.

The server may advertise zero or more complete segments.

# Drawbacks
[drawbacks]: #drawbacks

- This feature will require significant codebase changes to Hls.js, and may introduce regressions.
- The proposed changes may make the codebase more complicated. Hls.js has an event-based architecture, in which events follow a sequential pattern (e.g. LOADING -> LOADED -> PARSED -> BUFFERED). LHLS will require events to be fired out of regular order (e.g. LOADING -> LOADING -> LOADED -> LOADING -> PARSED -> LOADED). A poor implementation may make codebase maintenance more difficult.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

This section details the *why* of some of our design choices, and the information used to arrive at that conclusion. Often the choice of X over Y can be traced back to one or more of the guiding principles of this proposal.

## HTTP vs WebRTC

HTTP has been chosen as the preferred protocol because of it's scalability and ease of implementation. While WebRTC is faster, HTTP performance is good enough to achieve our goal of ~2s latency. The section below outlines the common arguments for and against each. Please note that our initial choice does not preclude the possibility of supporting WebRTC in the future. Much of the LHLS code which will be built into Hls.js is protocol-agnostic and will be reusable if and when WebRTC is built.

WebSockets are not under consideration as two-way communication has no current use, and provides no performance benefit over HTTP or WebRTC. 

#### HTTP

HTTP low-latency streaming allows developers to leverage existing HTTP video delivery infrastructure. The most notable benefit is that HTTP streams can be edge cached, allowing a live event to efficiently scale to large audiences. HTTP is also more straightforward to implement in Hls.js. We already have well-functioning code for handling HTTP requests, and domain knowledge which will help us implement LHLS. More developers server streams via HTTP instead of WebRTC and lends accessibility and adoption to our proposal.

HTTP uses TCP, and TCP has inherent latency which is makes it less well suited for streaming data compared to WebRTC. TCP has mechanisms such as flow control, retransmission, and error-free data transfer which are useful for serving sites but are often unnecessary for delivering video. These transfer mechanism each have a latency associated with them which repeats for each request. LHLS mitigates some of this latency of TCP by initiating the prefetch request before the segment is ready.

#### WebRTC

WebRTC relies on UDP for data delivery. UDP achieves significantly lower latency than TCP because it is lossy - it does not spend time guaranteeing the transmission of data. Lost data in a media stream is usually not fatal and results in dropped frames. WebRTC has been shown to achieve sub-second streaming latency.

WebRTC suffers from scalability. Each "peer", in this case the client, requires a direct connection to another peer. Streaming servers are able to act as peers but must serve each request with a unique connection. This requirement precludes edge caching. A fitting analogy is that WebRTC streams require a unique URL for every client. WebRTC does have the advantage that peers may also be clients but this requires additional code.

WebRTC also is less accessible than HTTP. Most developers today do not stream via WebRTC, and may not have the knowledge or resources to do so. Client support for WebRTC is available but more limited than existing HTTP clients. SSAI solutions are possible but also require further changes, and the breadth of support is not clear.

## Method of Advertising LHLS Segments

#### In-Playlist Signaling

In-playlist signaling happens when prefetch segments are advertised within the same playlist as complete segments. Prefetch segments are included at the bottom and are differentiated via the `EXT-X-PREFETCH` tag. This is the scheme outlined in this proposal. 

We chose this scheme because it is backwards compatible with clients which have not implemented this proposal. According to the HLS specification clients must ignore any tags not outlined within; since `EXT-X-PREFETCH` is not an official tag clients will be left with a standard HLS playlist. This allows a server to deliver a single manifest which plays in both LHLS and unsupported clients. This scheme also allows clients to disable or enable LHLS while the client is streaming. Developers may make this option available to the user so that they are able to choose an experience most beneficial to them.

Another benefit of this scheme is that it more easily takes advantage of the HLS specification. In most cases an in-playlist prefetch segments can be treated like a complete segments, which promotes code reuse and eases implementation. As an example a prefetch segment's Media Sequence Number is calculated relative to the complete segments in the playlist; if prefetch segments were elsewhere a new solution would have to be devised.

#### All-Playlist Signaling

All-Playlist signaling is when every Media Playlist contains only prefetch segments. In this scheme the `EXT-X-PREFETCH` tag is not required; all segments are assumed to be prefetch. This solution has the benefit of requiring little or no modifications from the HLS specification (what changes, if any, are hard to determine without actually making such a proposal). But this solution is not backwards compatible with clients not supporting this proposal.

#### Separate-Playlist Signaling

Separate-Playlist signaling is when a Master Playlist advertises an entire rendition as prefetch. Like all-playlist signaling, this scheme benefits from requiring little or no changes to the HLS specification on Media Playlists; it also has the added benefit of building backwards compatibility (contingent on an implementation which causes non-participating clients to ignore the rendition). This scheme, however, is more complex to implement in Hls.js. Hls.js works within the context of a single playlist - meaning that it chooses segments and gathers information from only one rendition. Supporting a second concurrent Media Playlist would most likely require significant amounts of new code as well as code refactored. Both servers and clients would also need to build a mechanism to synchronize properties such as Media Sequence Number and Discontinuity Sequence Numbers across complete and prefetch renditions.

## What is the cost of not doing this?

Low-latency streaming is becoming increasingly desired. Larger companies such as Akamai, Wowza, and Periscope offer low-latency streaming solutions. However, these solutions are not free or open. Not implementing this feature in Hls.js would make it more difficult for all video developers to access an easy and effective LHLS solution; if LHLS becomes a basic feature in the future, not having a FOSS solution will drive people towards a closed solution. Furthermore, developing LHLS in the open allows everyone to participate in its development.

# Supported Environments
[supported-environments]: #supported-environments

Only browsers which support the Fetch API with streaming body responses are able to support LHLS. This includes:

- Chrome >= 63
- Edge >= 16
- Safari >= 11.1
- Firefox >= 57 (streaming Fetch is currently behind a flag)

# Prior art
[prior-art]: #prior-art

- https://medium.com/@periscopecode/introducing-lhls-media-streaming-eb6212948bef
- https://speakerdeck.com/stswe/cmaf-low-latency-streaming-by-will-law-from-akamai
- https://www.wowza.com/products/capabilities/low-latency

# Unresolved questions
[unresolved-questions]: #unresolved-questions

## What parts of the design do you expect to resolve through the RFC process before this gets merged?

- SSAI Support
    - SCTE-35
- Exclusive support for CMAF (aka forbidding MPEG-TS)
- Should the manifest only update after the prefetch frags have completed? Can prefetch frags be repeated if they are not yet completed?
- Should playback rate and rebuffer handling be unified in this spec, or should it be up to the client implement as they see fit?
 
## What parts of the design do you expect to resolve through the implementation of this feature before stabilization?

Aside from `DISCONTINUITY`, It is not certain which tags need a prefetch prefix to ensure backwards compatibility with players that do not support this proposal. For example, we do not know how Safari's native player will handle a rogue `EXT-X-PROGRAM-DATE-TIME` tag. When test manifest are acquired, implementers must ensure that no unexpected behavior is introduced. 

## What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

- Alternative connection protocols (WebRTC, Websockets, etc.)
- Manifestless mode

# Examples
[examples]: #examples

The example below is a Media Playlist listing two available segments and two prefetch segments.

`URL: https://foo.com/bar.m3u8`
```
#EXTM3U
#EXT-X-VERSION:3
#EXT-X-TARGETDURATION:2
#EXT-X-MEDIA-SEQUENCE: 0
#EXT-X-DISCONTINUITY-SEQUENCE: 0
#EXT-X-PROGRAM-DATE-TIME:2018-09-05T20:59:06.531Z
#EXTINF:2.000
https://foo.com/bar/0.ts
#EXT-X-PROGRAM-DATE-TIME:2018-09-05T20:59:08.531Z
#EXTINF:2.000
https://foo.com/bar/1.ts

#EXT-X-PREFETCH:https://foo.com/bar/2.ts
#EXT-X-PREFETCH:https://foo.com/bar/3.ts
```

Segments 2 and 3 have a Media Sequence Number of 2 and 3, respectively. The next refresh of the Media Playlist transforms the oldest prefetch segment, and appends a new one:

```
#EXTM3U
#EXT-X-VERSION:3
#EXT-X-TARGETDURATION:2
#EXT-X-MEDIA-SEQUENCE: 1
#EXT-X-DISCONTINUITY-SEQUENCE: 0
#EXT-X-PROGRAM-DATE-TIME:2018-09-05T20:59:08.531Z
#EXTINF:2.000
https://foo.com/bar/1.ts
#EXT-X-PROGRAM-DATE-TIME:2018-09-05T20:59:10.531Z
#EXTINF:2.000
https://foo.com/bar/2.ts

#EXT-X-PREFETCH:https://foo.com/bar/3.ts
#EXT-X-PREFETCH:https://foo.com/bar/4.ts
```

Discontinuities:

```
#EXTM3U
#EXT-X-VERSION:3
#EXT-X-TARGETDURATION:2
#EXT-X-MEDIA-SEQUENCE: 0
#EXT-X-DISCONTINUITY-SEQUENCE: 0
#EXT-X-PROGRAM-DATE-TIME:2018-09-05T20:59:06.531Z
#EXTINF:2.000
https://foo.com/bar/0.ts
#EXT-X-PROGRAM-DATE-TIME:2018-09-05T20:59:08.531Z
#EXTINF:2.000
https://foo.com/bar/1.ts

#EXT-X-PREFETCH-DISCONTINUITY
#EXT-X-PREFETCH:https://foo.com/bar/5.ts
#EXT-X-PREFETCH:https://foo.com/bar/6.ts
```

In this case `5.ts` and `6.ts` have a Discontinuity Sequence Number of 1. On refresh:

```
#EXTM3U
#EXT-X-VERSION:3
#EXT-X-TARGETDURATION:2
#EXT-X-MEDIA-SEQUENCE: 1
#EXT-X-DISCONTINUITY-SEQUENCE: 0
#EXT-X-PROGRAM-DATE-TIME:2018-09-05T20:59:08.531Z
#EXTINF:2.000
https://foo.com/bar/1.ts
#EXT-X-DISCONTINUITY
#EXT-X-PROGRAM-DATE-TIME:2018-09-05T21:59:10.531Z
#EXTINF:2.000
https://foo.com/bar/5.ts

#EXT-X-PREFETCH:https://foo.com/bar/6.ts
#EXT-X-PREFETCH:https://foo.com/bar/7.ts
```

`5.ts, 6.ts, and 7.ts` all have the Discontinuity Sequence Number of 1. Note how the `PREFETCH-DISCONTINUITY` transformed to the conventional `EXT-X-DISCONTINUITY` tag, and how that tag still applies to prefetch segments.