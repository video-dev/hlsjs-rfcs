- Feature Name: Advanced Segment Downloader
- Start Date: 2018-08-18
- RFC PR: (leave this empty)
- Hls.js Issue: (leave this empty)

# Summary
[summary]: #summary

Low-latency HLS manifests advertise segments before the server transcodes them. If the LHLS-enabling metadata flag is present in the master manifest, Hls.js will initiate HTTP requests against each segment in a child rendition using chunked transfer encoding.

# Motivation
[motivation]: #motivation

Advanced segment downloading enables Hls.js to play low-latency HLS streams. This RFC is a component of the LHLS RFC, please see {LINK} for more details.

During a live stream, current transcoders wait until several segments are finished until advertising a new manifest; this incurs a latency equal to the total duration of segments plus the time to transcode them. Low-latency transcoders advertises segments before being available, and pushes data to the segment endpoint in chunks during transcoding. This reduces the delay equal to the duration of the first segment's chunk plus the time to transcode it.

Low-latency transcoders want an HLS client which takes advantage of early segment advertising. Hls.js, being the most widely used web HLS client, is in a position to enable all developers access to a FOSS LHLS client.

# High-Level Explanation
[high-level-explanation]: #high-level-explanation

A low-latency manifest is signaled via metadata in the master manifest. When Hls.js encounters this tag, it enters low-latency mode. In low latency mode, Hls.js instructs it's fragment loader to initiate a connection to each segment in the manifest. Throughout the live stream Hls.js continuously re-downloads the manifest and initiates connections against any new segments. Hls.js uses the Fetch API with a streaming body response; each segment download is received as multiple smaller chunks.

The advanced segment downloading system tracks the bandwidth of the client's connection and informs the ABR algorithm so that it may choose the most optimal rendition.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

#### Low-latency activation
The non-standard `EXT-X-LOWLATENCY` tag will put Hls.js into low-latency mode.


#### Advanced Segment Downloading



#### Streaming Body Fetch


#### Bandwidth Tracking


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

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?
