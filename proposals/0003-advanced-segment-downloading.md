- Feature Name: Advanced Segment Downloader
- Start Date: 2018-08-18
- RFC PR: (leave this empty)
- Hls.js Issue: (leave this empty)

# Summary
[summary]: #summary

Low-latency transcoding pipelines advertise segments in their manifest before that segment is actually transcoded. Using chunked transfer encoding, Hls.js will open an HTTP connection to each segment within the manifest. The transcoder will push partial segments as they're being processed down this connection; when received, Hls.js pushes and each partial segment through the progressive streaming engine so that it may be immediately buffered.

# Motivation
[motivation]: #motivation

During a live stream, current transcoders wait until several segments are finished until advertising a new manifest; this incurs a latency equal to the total duration of segments plus the time to transcode them. Low-latency transcoders advertises segments before being available, and pushes data to the segment endpoint in chunks during transcoding. This reduces the delay equal to the duration of the first segment's chunk plus the time to transcode it.

Further contributing to latency are delays intrinsic to TCP. By requesting the segment before it's needed Hls.js can eliminate latency added by the initial TCP handshake and roundtrip.

# Guide-Level Explanation
[Guide-level-explanation]: #guide-level-explanation


Low-latency segments are advertised via the `EXT-X-PREFETCH` metadata tag in the child manifest. These segments are in the process of transcoding and are the closest content to capture. If Hls.js is on the edge of a live stream, it will initiate downloading on these segments - in turn, the server will respond with partial chunks of the each segment as it is ready. Using the Fetch API with streaming body responses Hls.js can receive these partial segments, transmux, and buffer them. In this way Hls.js is able to buffer and play content with significantly reduced latency to the live edge.

The structure of a low-latency manifest allows Hls.js to control how and when low-latency segments are used. LHLS manifests advertise already-transcoded segments in addition to prefetched ones. If Hls.js is not on the live edge it may not choose to use them; and in browsers which do not support the APIs needed for LHLS these segments can be safely ignored.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation





# Drawbacks
[drawbacks]: #drawbacks

Increased complexity and added code with no benefit for the non-low-latency case. LHLS is expected to represent a minority of media played by Hls.js.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

# Supported Environments
[supported-environments]: #supported-environments

Only browsers which support the Fetch API with streaming body responses are able to support progressive parsing.

- Chrome >= 63
- Edge >= 16
- Safari >= 11.1
- Firefox >= 57 (only behind a flag)

# Prior art
[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- Does this feature exist in another player?
- Has this feature been discussed in a blog post, paper, or presentation?

This section is intended to encourage you as an author to think about the lessons from other players, and provide readers of your RFC with a fuller picture.
If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other languages.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What to do when encountering an HTTP error?
- Should a connection ever time out? If so, how long is this timeout? 
