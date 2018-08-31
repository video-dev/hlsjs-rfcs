mux notes:

- No SSAI
- Need ABR
    - Segment download time is always segment time
    - Periscope LHLS video tall
    - Not using 100% bandwidth with LHLS
    - Heuristic approach
    - Probe already downloaded segment
    - Test things already transcoded
    - Manifest download
    - Try to download & see if you can do it without rebuffering
- How to catch back up to the live edge
    - Modify playback rate
    - Drop frames & force catchup
    - Being on the live edge is the priority
- Metrics will look like trash
    - Add latency as a metric so we can show that the score isn't bad because you care about latency
- QUIC/HTTP2 + LHLS
    - TCP ramp-up is a problem