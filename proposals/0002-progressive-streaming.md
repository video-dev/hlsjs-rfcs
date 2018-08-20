- Feature Name: Progressive Streaming Pipeline
- Start Date: 2018-08-18
- RFC PR: (leave this empty)
- Hls.js Issue: (leave this empty)

# Summary
[summary]: #summary

At a high level, Hls.js' streaming pipeline performs four steps: download, demux, remux, and buffer. Hls.js completes each step sequentially and with a whole segment. However, the whole segment isn't required for part of it to be pipelined - each segment contains several GOPs (group of pictures), and each GOP can be independently transmuxed and buffered. This RFC proposes modifications to Hls.js' streaming pipeline so that it may operate on partial segments, therefore reducing the latency from download to first buffer.

# Motivation
[motivation]: #motivation

The introduction of a progressive streaming pipeline in Hls.js would reduce latency not only for live but for VOD. Progressively buffering segments allows Hls.js to reduce it's TTFF (time to first frame), and by the same token reduce the rebuffering time caused by network instability.

A progressive streaming engine would enable Hls.js to play low-latency HLS streams. This RFC is a component of the LHLS RFC, please see {LINK} for more details.


# guide-Level Explanation
[guide-level-explanation]: #guide-level-explanation

The advanced segment download {link} will provide the progressive streaming pipeline with chunks of segments. Hls.js passes each chunk to the demuxer, which scans the partial segment for complete .ts packets (at 188-byte intervals). The demuxer parses through each .ts packet and extracts metadata and samples; when finished, if samples were found, they are passed to the remuxer. Partial packets (< 188 bytes) are saved for the next iteration. This process continues until the segment download completes at which point the demuxer will flush. The remuxer takes each set of samples and boxes them into fragmented MP4s. Each .fmp4 fragment is then appended to an MSE sourcebuffer.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

#### Downloading

#### Demuxing

#### Remuxing

#### Appending

# Drawbacks
[drawbacks]: #drawbacks

- The codebase becomes more complicated. Hls.js' event-based architecture must now tolerate a non-sequential ordering. A poor architecture would make managing and updating state difficult.
- Because of the new event system, developer integrations will need to be updated. This will be a major release which will introduce several breaking changes.
- Currently, the Fetch API cannot cancel requests. Hls.js cancels downloads to save bandwidth when switching renditions.
- Developers not wanting progressive streaming (probably) be able to disable this feature. However, the added features would increase the size of their bundle without added benefit.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Developers could lower the latency of the sequential pipeline through smaller segments . However, this incurs a higher transcoding cost (more iframes).

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

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?
