---
title: "MOQT Extension for VOD with Live Transitions"
abbrev: "moqt-vodl"
category: std
ipr: trust200902

docname: draft-frindell-moq-vod-live-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Web and Internet Transport"
workgroup: "Media Over QUIC"
keyword:
 - moq
 - vod
 - video on demand
venue:
  group: "Media Over QUIC"
  type: "Working Group"
  mail: "moq@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/moq/"
  github: "afrind/draft-frindell-moq-vod-live"
  latest: "https://afrind.github.io/draft-frindell-moq-vod-live/draft-frindell-moq-vod-live.html"

stand_alone: yes
smart_quotes: no
pi: [toc, sortrefs, symrefs]

author:
  -
    fullname: Alan Frindell
    organization: Meta
    email: afrind@meta.com

normative:
  MOQT: I-D.ietf-moq-transport

informative:

---

--- abstract

This document defines an extension to Media over QUIC Transport (MOQT)
that enables efficient delivery of pre-recorded (Video On Demand) content.
The extension introduces subscriber-paced delivery, refined timeout
semantics for prioritized subgroup delivery, and mechanisms for clean
transition from VOD to live playback.

--- middle

# Introduction

MOQT {{MOQT}} is designed primarily for live media delivery, where content
is published in real-time and subscribers consume it as it arrives. Video
On Demand (VOD) presents different requirements: content is already
available, and the subscriber controls consumption rate.

This extension enables MOQT to efficiently serve VOD content by:

1. Allowing subscribers to request past content via SUBSCRIBE with
   pacing control
2. Delivering content across multiple subgroup streams (unlike FETCH
   which uses a single stream)
3. Providing prioritized delivery with refined timeout semantics
4. Enabling clean transitions from VOD to live playback

## Conventions and Definitions

{::boilerplate bcp14-tagged}

## Terminology

Base Layer:
: The content at the highest priority (lowest Publisher Priority value)
  within a group. Base layer content is essential for playback and
  receives preferential treatment in timeout and delivery decisions.

Enhancement Layer:
: Any content at lower priority (higher Publisher Priority value) than
  the base layer within the same group. Enhancement layer content
  improves quality but can be dropped under constrained conditions.

# Extension Negotiation {#negotiation}

Endpoints negotiate support for this extension during session setup.

A client that supports the VOD extension includes the SUPPORTS_VOD
parameter in CLIENT_SETUP:

~~~
SUPPORTS_VOD Parameter {
  Type (vi64) = 0xVOD1,
  Length (vi64) = 1,
  Value (8) = 1
}
~~~

A server that supports the VOD extension includes the SUPPORTS_VOD
parameter in SERVER_SETUP with value 1.

A subscriber MUST NOT send a SUBSCRIBE with MODE=VOD unless the
publisher indicated support via SUPPORTS_VOD in SERVER_SETUP.

# SUBSCRIBE Parameters {#subscribe-params}

This extension defines the following parameters for use in SUBSCRIBE,
PUBLISH_OK, and REQUEST_UPDATE messages. These parameters allow the
subscriber to configure VOD delivery mode.

When the subscriber initiates a subscription via SUBSCRIBE, these
parameters are included directly. When the publisher initiates via
PUBLISH, the subscriber includes these parameters in PUBLISH_OK to
request VOD delivery.

## MODE Parameter {#mode-param}

The MODE parameter indicates the delivery mode for a subscription.

~~~
MODE Parameter {
  Type (vi64) = 0xVOD2,
  Length (vi64) = 1,
  Value (8)
}
~~~

Defined values:

| Value | Mode | Description |
|-------|------|-------------|
| 0x0   | LIVE | Default. Standard MOQT subscription behavior. |
| 0x1   | VOD  | Subscriber-paced delivery of past content. |

When MODE=VOD is specified:

1. The relay MUST NOT coalesce this subscription with other
   subscriptions, even if the Track and filter match an existing
   subscription.

2. The publisher MUST begin delivery from the specified Start filter,
   even if the content is in the past.

3. Delivery uses multiple subgroup streams, as with normal SUBSCRIBE,
   not a single stream as with FETCH.

4. The subscriber controls pacing via GROUP_INTERVAL ({{group-interval-param}}).

5. All objects in the requested range are delivered (fill semantics).

## GROUP_INTERVAL Parameter {#group-interval-param}

The GROUP_INTERVAL parameter specifies the minimum time between starting
delivery of consecutive groups.

~~~
GROUP_INTERVAL Parameter {
  Type (vi64) = 0xVOD3,
  Length (vi64),
  Value (vi64)
}
~~~

The value is expressed in milliseconds.

For example, a track with 2-second groups played at 1x speed would use
GROUP_INTERVAL = 2000. The same track at 1.5x speed would use
GROUP_INTERVAL = 1333.

If a subscriber sends a GROUP_INTERVAL parameter without MODE=VOD, the
publisher MUST respond with REQUEST_ERROR.

If GROUP_INTERVAL is not specified by the subscriber, the publisher
uses the Track's GROUP_INTERVAL extension ({{group-interval-ext}})
if present, otherwise delivers as fast as possible.

## START_GROUP_OFFSET Parameter {#start-group-offset-param}

The START_GROUP_OFFSET parameter optionally specifies a start location
relative to the current live edge, expressed as a number of groups
behind live.

~~~
START_GROUP_OFFSET Parameter {
  Type (vi64) = 0xVOD7,
  Length (vi64),
  Value (vi64)
}
~~~

The value is the number of groups behind the live edge to begin
delivery. A value of 0 means start at the current live group.

When START_GROUP_OFFSET is present, it overrides any Start filter in the
SUBSCRIBE message. The publisher calculates the actual start group as:

~~~
start_group = live_edge_group - START_GROUP_OFFSET
~~~

If the calculated start_group is less than the first available group,
delivery begins from the first available group.

This parameter is OPTIONAL. When not present, the Start filter from
the SUBSCRIBE message is used directly.

### Examples

~~~
START_GROUP_OFFSET = 0    // Start at current live group
START_GROUP_OFFSET = 150  // Start 150 groups behind live (5 min at 2s groups)
START_GROUP_OFFSET = 900  // Start 900 groups behind live (30 min at 2s groups)
~~~

If a subscriber sends START_GROUP_OFFSET without MODE=VOD, the publisher
MUST respond with REQUEST_ERROR.

## MAX_SUBSCRIPTION_SUBGROUPS Parameter {#max-sub-subgroups-param}

The MAX_SUBSCRIPTION_SUBGROUPS parameter specifies the initial limit on
the number of subgroup streams that can be opened for this
subscription.

~~~
MAX_SUBSCRIPTION_SUBGROUPS Parameter {
  Type (vi64) = 0xVOD8,
  Length (vi64),
  Value (vi64)
}
~~~

The value is the maximum subgroup count the publisher can open for
this subscription. The default value is 1.

This parameter provides subscription-level flow control, ensuring fair
sharing of QUIC stream resources across multiple subscriptions on the
same connection.

This parameter can be included in:

- SUBSCRIBE with MODE=VOD (subscriber initiates VOD subscription)
- PUBLISH_OK with MODE=VOD (subscriber accepts PUBLISH with VOD mode)
- REQUEST_UPDATE with MODE=VOD (either endpoint transitions to VOD mode)

For MODE=LIVE subscriptions, this parameter is ignored.

When a subscription transitions to VOD mode, the Subgroup Count resets
to 0 and the Max Subgroup Count is set from the parameter value. If
the parameter is not included, the default value of 1 is used.

The subscriber can increase this limit during the subscription by
sending MAX_SUBSCRIPTION_SUBGROUPS messages ({{max-sub-subgroups-msg}}).

# Control Messages {#control-messages}

This extension defines the following control messages.

## MAX_SUBSCRIPTION_SUBGROUPS Message {#max-sub-subgroups-msg}

The MAX_SUBSCRIPTION_SUBGROUPS message allows the subscriber to increase
the number of subgroup streams allowed for a subscription.

~~~
MAX_SUBSCRIPTION_SUBGROUPS Message {
  Type (vi64) = 0xVOD9,
  Length (vi64),
  Request ID (vi64),
  Max Subgroup Count (vi64)
}
~~~

* Request ID: Identifies the subscription.

* Max Subgroup Count: The new maximum subgroup count allowed.
  This value is cumulative (total ever allowed), not a delta. It MUST
  be greater than or equal to any previously sent value.

The subscriber sends this message to grant more subgroup capacity to
the publisher. The publisher MUST NOT open more subgroup streams than
the current Max Subgroup Count value.

### Flow Control Semantics {#subgroup-flow-control}

The publisher tracks two values per subscription:

* **Subgroup Count**: The total number of subgroup streams opened
  (including completed and reset streams).

* **Max Subgroup Count**: The current limit from the subscriber (initial
  parameter value or most recent MAX_SUBSCRIPTION_SUBGROUPS message).

The publisher MAY open a new subgroup stream only when:

~~~
Subgroup Count < Max Subgroup Count
~~~

When the publisher cannot open a subgroup due to this limit:

1. The subgroup is queued for delivery when capacity becomes available.

2. If DELIVERY_TIMEOUT is specified and another subgroup is waiting for
   the slot (contention), the timeout clock starts. If the timeout
   expires before the subgroup can be opened, the publisher skips it
   and moves to the next.

3. The base layer is exempt from timeout. If the base layer
   cannot be opened due to the limit, the publisher waits indefinitely.

The subscriber grants more capacity by sending MAX_SUBSCRIPTION_SUBGROUPS
with a higher value. A recommended approach: when the base layer
completes, set Max Subgroup Count to the initial value plus the total
number of subgroup streams received (completed or reset) so far. This
batches credit grants efficiently while ensuring the base layer is never
blocked by the limit.

# Track Extension Headers {#track-extensions}

This extension defines the following Track Extension Headers.

## GROUP_INTERVAL Extension {#group-interval-ext}

The GROUP_INTERVAL extension specifies the default interval between
groups, typically corresponding to the group duration at 1x playback
speed.

~~~
GROUP_INTERVAL Extension {
  Type (vi64) = 0xVOD3,
  Length (vi64),
  Value (vi64)
}
~~~

The value is expressed in milliseconds.

Publishers SHOULD include this extension on tracks containing timed
media. It informs subscribers of the intended playback rate and allows
relays to pace delivery when no subscriber override is present.

When a subscriber specifies GROUP_INTERVAL as a SUBSCRIBE parameter, it
overrides the Track extension value. This allows playback at speeds other
than 1x.

Relays serving MODE=VOD subscriptions can use any means to satisfy
downstream subscribers at their requested rates, including cache,
upstream MOQT subscriptions, HTTP retrieval, or other mechanisms. The
relay is not required to forward this extension unchanged.

# Object Extension Headers {#object-extensions}

This extension defines the following Object Extension Headers.

## LIVE_EDGE_DELTA Extension {#live-edge-delta}

The LIVE_EDGE_DELTA extension indicates how far behind the live edge
the current group is.

~~~
LIVE_EDGE_DELTA Extension {
  Type (vi64) = 0xVOD4,
  Length (vi64),
  Value (vi64)
}
~~~

The value is the number of groups between the current group and the
live edge. A value of 0 indicates the current group is at the live edge.

Publishers and relays MUST include this extension only on Object ID 0
(the first object) of each group delivered via MODE=VOD subscriptions
when the track is live.

Publishers and relays MUST NOT include this extension when:

- The subscription mode is LIVE
- The track is complete (no new content will be published)
- The Object ID is not 0

Subscribers use this extension to:

1. Determine how far behind live they are
2. Plan transition to live subscription
3. Adjust playback speed to catch up or maintain distance

This extension MUST NOT be cached or forwarded to another subscriber.

### Example

~~~
Group 4500, Object 0:
  LIVE_EDGE_DELTA = 502   // Live edge is at group 5002

Group 4501, Object 0:
  LIVE_EDGE_DELTA = 502   // Live edge advanced to 5003

Group 4998, Object 0:
  LIVE_EDGE_DELTA = 12    // Catching up

Group 5008, Object 0:
  LIVE_EDGE_DELTA = 2     // Almost live
~~~

# Delivery Semantics {#delivery}

## Fill Semantics

A MODE=VOD subscription has "fill" semantics similar to FETCH: all
objects in the requested range are delivered and/or retrieved. The
publisher is responsible for attempting delivery for every object
between the Start and End filters, retrieving from upstream or cache
as necessary. Gaps in delivery indicate objects that do not exist at
the original publisher, or objects that were dropped by the immediate
publisher due to congestion.


## Delivery Constraints {#delivery-constraints}

Content is organized by group, priority, and object ID. The following
constraints apply regardless of delivery mechanism:

1. **Within a priority level**: At most one group's content at that
   priority can be in-flight at a time. Within a group, content at the
   same priority is serialized by object ID. For subgroups, the first
   object ID in the subgroup is used for comparison.

2. **Across groups**: A publisher MUST NOT begin delivering content
   from group N+1 at priority P until all content from group N at
   priority P has completed or been abandoned.

3. **Base layer first**: Within a group, the publisher MUST NOT begin
   delivery of enhancement layers until the base layer has started.

Different priority levels operate independently and can be delivering
different groups concurrently.

## Pacing {#pacing}

When GROUP_INTERVAL is specified (via parameter or Track extension),
the publisher SHOULD wait at least GROUP_INTERVAL milliseconds between
starting delivery of consecutive groups.

The interval is measured from when delivery of group N begins (first
content at the base layer starts) to when delivery of group N+1 begins.

If delivery of a group takes longer than GROUP_INTERVAL (due to network
conditions or content size), the publisher begins the next group
immediately upon the base layer completing.

## Stream Delivery {#stream-delivery}

QUIC MAX_STREAMS provides connection-wide flow control, while
MAX_SUBSCRIPTION_SUBGROUPS ({{max-sub-subgroups-param}}) provides
subscription-level flow control. Both mechanisms work together to
manage stream resources.

When the publisher cannot open a new subgroup stream due to either
limit (QUIC MAX_STREAMS or MAX_SUBSCRIPTION_SUBGROUPS), the publisher
waits for the subscriber to grant more credit.

While waiting for stream credit:

1. The publisher queues the subgroup for delivery when credit becomes
   available.

2. If DELIVERY_TIMEOUT is specified, timeout behavior applies as
   described in {{timeout}}.

3. The subscriber grants more credit by consuming completed streams.
   As the publisher completes subgroup streams (sends FIN) and the
   subscriber consumes them, the subscriber sends MAX_STREAMS updates
   (for QUIC-level credit) and MAX_SUBSCRIPTION_SUBGROUPS messages
   (for subscription-level credit) to allow more streams to be opened.

This allows the subscriber to indirectly control delivery rate by
managing stream credit.

## Datagram Delivery {#datagram-delivery}

When objects are delivered via datagrams rather than streams, flow
control backpressure does not apply. Datagrams are sent immediately
without waiting for subscriber acknowledgment.

However, congestion control and send-side queueing still affect
datagram delivery. When the QUIC send buffer is full due to congestion,
datagrams can be queued.

The delivery constraints in {{delivery-constraints}} apply to
datagrams.  Additionally, DELIVERY_TIMEOUT applies to queued
enhancement layer datagrams.  If an enhancement layer datagram remains
in the send queue longer than DELIVERY_TIMEOUT due to congestion, the
publisher MAY drop the datagram rather than sending it late.

Note: Tracks designed for VOD delivery SHOULD NOT use datagrams for
base layer content. Datagrams are inherently unreliable and can be lost
due to network conditions, but VOD mode expects the base layer to be
delivered reliably. Datagrams might be appropriate for enhancement
layers where loss is acceptable.

### Example: Audio Track with Datagrams

Consider an audio track with 50 objects per 1-second group, each
delivered as a datagram, all at priority 64:

~~~
GROUP_INTERVAL = 1000ms (1x playback)
DELIVERY_TIMEOUT = 500ms

T=0ms:    Begin Group 0
          Send Object 0, then Object 1, then Object 2, ...
          (serialized within group, same priority)
T=1000ms: Begin Group 1
          G0 objects are sent/dropped first
          If G0 objects still queued > 500ms, drop them
T=2000ms: Begin Group 2
          ...
~~~

## Multi-Track Synchronization {#multi-track-sync}

There is no inherent synchronization between tracks delivered via
MODE=VOD. Each track is paced independently according to its
GROUP_INTERVAL parameter.

Subscribers are responsible for aligning delivery across tracks. This
can be achieved by:

- **Adjusting GROUP_INTERVAL**: Set different GROUP_INTERVAL values
  per track to align their delivery rates based on group durations.

- **Pause and resume**: Use REQUEST_UPDATE with Forward=0/1 to pause
  one track while another catches up.

- **START_GROUP_OFFSET**: Begin tracks at different offsets to align
  their starting points based on media timestamps.

For example, if video has 2-second groups and audio has 1-second groups,
setting video GROUP_INTERVAL=2000 and audio GROUP_INTERVAL=1000 delivers
both at 1x realtime, but the subscriber might need to pause one track
if they drift apart due to network conditions.

## Timeout Semantics {#timeout}

DELIVERY_TIMEOUT, when used with MODE=VOD, has refined semantics to
enable prioritized delivery under constrained conditions. This section
describes timeout behavior for stream-based delivery; datagram timeout
behavior is described in {{datagram-delivery}}.

### Timeout Activation {#timeout-activation}

The DELIVERY_TIMEOUT clock for content at a priority level starts only
when there is contention - that is, when other content needs the slot
that the current content is occupying.

Contention occurs in two scenarios:

1. **Within a group**: Content at the same priority level is serialized
   by first object ID. If content A is in-flight and content B (same
   priority, higher first object ID) is ready to start, B is blocked
   by A. Timeout starts on A.  This does not apply within a single
   subgroup.

2. **Across groups**: Content from group N+1 at priority P is ready
   to start, but content from group N at priority P is still in
   flight. Timeout starts on the group N content.

If nothing is waiting for the slot, the in-flight content continues
without timeout pressure.

When a new group begins:

1. The publisher begins delivery of the new group's base layer.

2. Enhancement layers of the new group wait for their priority level
   to become available (no in-flight content at that priority from the
   previous group).

### Base Layer Immunity {#base-immunity}

The base layer within each group MUST NOT be timed out. It
contains essential content required for playback.

If the base layer cannot be delivered within available bandwidth, the
subscriber SHOULD switch to a lower-quality track.

### Timeout Behavior {#timeout-behavior}

When a stream's timeout expires:

1. The publisher resets the stream
2. When all content at that priority level has completed or been
   abandoned, the priority level becomes available for the next group
3. The publisher can begin content at that priority for the next group

### Example {#timeout-example}

Consider a track with 2 subgroups per group: SG0 (priority 0, base) and
SG1 (priority 128, enhancement). GROUP_INTERVAL = 2000ms,
DELIVERY_TIMEOUT = 1000ms.

~~~
T=0ms:    Begin Group 0
          Start G0/SG0 (base)
          Start G0/SG1 (enhancement) -- allowed, base has started

T=1500ms: G0/SG0 completes
          G0/SG1 still in progress (slow)

T=2000ms: GROUP_INTERVAL elapsed, begin Group 1
          Start G1/SG0 (base) -- SG0 slot is free
          G1/SG1 wants to start but G0/SG1 occupies the slot
          Start timeout clock on G0/SG1 (contention)

T=3000ms: G0/SG1 timeout expires
          Reset G0/SG1 stream
          Start G1/SG1 (enhancement) -- SG1 slot now free
          G1/SG0 completes

T=4000ms: GROUP_INTERVAL elapsed, begin Group 2
          Start G2/SG0 (base)
          G2/SG1 wants to start but G1/SG1 occupies the slot
          Start timeout clock on G1/SG1 (contention)
          ...
~~~

In this example, the base layer (SG0) flows continuously at the
requested rate. The enhancement layer (SG1) is reset when it cannot
keep up, resulting in degraded but continuous playback.

## Backpressure {#backpressure}

When the subscriber is reading slowly (due to playback buffering,
application processing, or network congestion), QUIC flow control
blocks individual streams.

For MODE=VOD subscriptions without DELIVERY_TIMEOUT, the publisher
waits indefinitely for the subscriber to consume data.

When DELIVERY_TIMEOUT is specified, timeout behavior applies as
described above. Timeouts operate independently per priority level -
a blocked base layer does not suspend enhancement layer timeouts.

## Pause and Resume {#pause-resume}

Subscribers can pause and resume VOD delivery at any time using
REQUEST_UPDATE with the Forward parameter:

- Forward = 0: Pause delivery. The publisher stops sending new objects
  but does not reset in-flight streams.  All delivery timeout timers are
  paused.

- Forward = 1: Resume delivery. The publisher continues from where it
  paused and restarts any delivery timers.

This allows subscribers to implement pause/play functionality without
tearing down the subscription, and to manage buffer depth dynamically.

## Catching Up After Pause {#catch-up-after-pause}

When a live subscription is paused with REQUEST_UPDATE Forward=0, the
subscriber falls behind the live edge. Upon resuming, the subscriber
might wish to catch up to live at an accelerated rate.

To catch up, the subscriber sends REQUEST_UPDATE with:

~~~
REQUEST_UPDATE {
  Request ID,
  Forward = 1,
  Parameters:
    MODE = VOD
    START_GROUP_OFFSET = 0              // Resume from current position
    GROUP_INTERVAL = <accelerated rate>  // Optional
    MAX_SUBSCRIPTION_SUBGROUPS = N       // Optional
}
~~~

The START_GROUP_OFFSET parameter specifies where to resume:

- START_GROUP_OFFSET = 0: Resume from the current position (typical case)
- START_GROUP_OFFSET = N: Restart N groups behind live

This switches the subscription to VOD mode, enabling:

- Fill semantics for the backlog of content
- Paced delivery at the specified GROUP_INTERVAL (faster than realtime)
- Delivery constraints and timeout behavior as specified in
  {{delivery-constraints}} and {{timeout}}
- Subgroup flow control with count reset (see {{max-sub-subgroups-param}})

When the subscriber catches up to the live edge, the publisher sends
REQUEST_UPDATE with MODE=LIVE as described in {{publisher-transition}},
and the subscription returns to live delivery.

### Example Flow

~~~
Subscriber                         Publisher
    |                                  |
    |  SUBSCRIBE (live)                |
    |--------------------------------->|
    |  SUBSCRIBE_OK                    |
    |<---------------------------------|
    |                                  |
    |  Objects (live, group 1000)      |
    |<---------------------------------|
    |                                  |
    |  REQUEST_UPDATE (Forward=0)      |  // Pause
    |--------------------------------->|
    |                                  |
    | ... time passes, live at 1050 ...|
    |                                  |
    |  REQUEST_UPDATE                  |  // Resume with catch-up
    |    Forward=1                     |
    |    MODE=VOD                      |
    |    START_GROUP_OFFSET=0          |  // From current position
    |    GROUP_INTERVAL=500            |  // 4x speed for 2s groups
    |--------------------------------->|
    |                                  |
    |  Objects (VOD mode, paced fast)  |
    |<---------------------------------|
    |  Groups 1001-1049 at 4x speed    |
    |                                  |
    |  REQUEST_UPDATE                  |  // Caught up
    |    MODE=LIVE                     |
    |    LARGEST_LOCATION={1050,x}     |
    |<---------------------------------|
    |                                  |
    |  REQUEST_OK                      |
    |--------------------------------->|
    |                                  |
    |  Objects (live mode)             |
    |<---------------------------------|
~~~

If GROUP_INTERVAL is omitted when resuming with MODE=VOD, the publisher
delivers the backlog as fast as possible (no pacing).

If START_GROUP_OFFSET is omitted, the subscriber resumes from their
current position (equivalent to START_GROUP_OFFSET=0).

## Delivery Examples {#delivery-examples}

This section illustrates how delivery constraints, flow control, timeout,
and credit granting work together.

### Video Track Example

Consider a video track with 2 subgroups per group at different priorities:

- SG0 at priority 0 (base layer)
- SG1 at priority 128 (enhancement layer)

Since they have different priorities, SG0 and SG1 can be in-flight
concurrently. However, G0/SG0 completes before G1/SG0 starts, and
G0/SG1 completes before G1/SG1 starts (per-priority ordering).

### Audio Track Example

Consider an audio track with 50 subgroups per group, all at priority 64:

- SG0-SG49 all at priority 64, ordered by first object ID

Within Group 0, subgroups are serialized: SG0 completes, then SG1
starts, then SG2, etc. Only after all 50 subgroups of Group 0 complete
(or are reset) can Group 1's subgroups begin.

### Combined Flow Control and Credit Example

~~~
Setup:
  Track has 2 subgroups: base (pri 0), enhancement (pri 128)
  Initial MAX_SUBSCRIPTION_SUBGROUPS = 4
  GROUP_INTERVAL = 2000ms
  DELIVERY_TIMEOUT = 1000ms
  Enhancement layer is slow

Publisher state: Count=0, Max=4, Received=0

T=0ms: Begin Group 0
  Open G0/base → Count=1
  Open G0/enh → Count=2

T=1500ms:
  G0/base completes
  Received=1 → Subscriber sends Max = 4+1 = 5

T=2000ms: Begin Group 1
  Open G1/base → Count=3
  G1/enh blocked by G0/enh (ordering constraint)
  G0/enh slow → timeout clock starts (contention)

T=3000ms: G0/enh timeout expires
  Reset G0/enh, Received=2
  Open G1/enh → Count=4

T=3500ms:
  G1/base completes
  Received=3 → Subscriber sends Max = 4+3 = 7

T=4000ms: Begin Group 2
  Open G2/base → Count=5
  G2/enh blocked by G1/enh → timeout on G1/enh

T=5000ms: G1/enh timeout expires
  Reset G1/enh, Received=4
  Open G2/enh → Count=6

...pattern continues...

Key observations:
- Base layer flows continuously (never blocked)
- Enhancement times out due to ordering contention
- Credit granted on base completion includes all received streams
- Max always stays ahead of Count
~~~

# VOD to Live Transition {#transition}

When a MODE=VOD subscription reaches the live edge (delivers the current
Largest object), the publisher can transition the subscription to live
mode, or the subscriber can manage the transition explicitly.

## Publisher-Initiated Transition {#publisher-transition}

When the publisher has delivered the Largest Object (the most recent
object available at the time), it sends a REQUEST_UPDATE to signal
the transition from VOD to live mode:

~~~
REQUEST_UPDATE {
  Request ID,
  Parameters:
    MODE = LIVE (0x0)
    LARGEST_LOCATION = {Group, Object}
}
~~~

The LARGEST_LOCATION parameter indicates the last object delivered
with fill semantics. All objects up to and including this location
were delivered (or explicitly reset due to timeout). Objects after
this location will be delivered with live semantics as they are
published.

Upon receiving this REQUEST_UPDATE, the subscriber:

- Sends REQUEST_OK to acknowledge the transition and continue with
  live delivery, OR
- Sends UNSUBSCRIBE if it only wanted the VOD content

After the subscriber sends REQUEST_OK, the subscription continues
with normal live delivery semantics. GROUP_INTERVAL no longer applies,
and the publisher delivers new objects as they are published.

### Example Flow

~~~
Subscriber                         Publisher
    |                                  |
    |  SUBSCRIBE                       |
    |    MODE=VOD                      |
    |    START_GROUP_OFFSET=150        |
    |    GROUP_INTERVAL=1333           |
    |--------------------------------->|
    |                                  |
    |  SUBSCRIBE_OK                    |
    |<---------------------------------|
    |                                  |
    |  Objects (fill mode, paced)      |
    |<---------------------------------|
    |  ...                             |
    |  Group 5000, LIVE_EDGE_DELTA=0   |
    |<---------------------------------|
    |                                  |
    |  REQUEST_UPDATE                  |
    |    MODE=LIVE                     |
    |    LARGEST_LOCATION={5000,15}    |
    |<---------------------------------|
    |                                  |
    |  REQUEST_OK                      |
    |--------------------------------->|
    |                                  |
    |  Objects (live mode)             |
    |<---------------------------------|
~~~

## Subscriber-Initiated Transition {#subscriber-transition}

Alternatively, subscribers can manage the transition explicitly by
monitoring LIVE_EDGE_DELTA and issuing a separate live subscription
before the VOD subscription completes.

### Procedure

1. Monitor LIVE_EDGE_DELTA on each group's first object
2. When LIVE_EDGE_DELTA approaches a threshold (e.g., 5 groups):
   - Calculate the expected group where VOD will end
   - Issue a new SUBSCRIBE (MODE=LIVE) starting after that group
   - Send a REQUEST_UPDATE ending the VOD subscription before that group
3. Continue playback from the live subscription

This approach allows subscribers to control the exact transition point
and ensures no gap in delivery.

## Pure VOD Content {#pure-vod}

When the track is complete (no new content will be published), the
publisher does not send REQUEST_UPDATE with MODE=LIVE. Instead, the
subscription ends normally with PUBLISH_DONE when all content has
been delivered.

## Using SWITCH

If the SWITCH mechanism is supported, subscribers can use it to
atomically transition from VOD to live without managing two
subscriptions.

# Security Considerations {#security}

## Resource Exhaustion

MODE=VOD subscriptions are not coalesced, which could allow malicious
subscribers to exhaust relay resources by issuing many VOD subscriptions
to the same track with different start points.

Relays SHOULD implement rate limiting on VOD subscriptions and MAY
reject excessive subscriptions with an appropriate error code.

## Timing Attacks

The LIVE_EDGE_DELTA extension reveals information about publisher
activity. In scenarios where this is sensitive, publishers MAY choose
not to include this extension, at the cost of subscribers being unable
to anticipate the live transition.

# IANA Considerations {#iana}

## MOQT Parameters Registry

This document registers the following entries in the "MOQT Parameters"
registry:

| Value  | Parameter Name | Message Types | Specification |
|--------|----------------|---------------|---------------|
| 0xVOD1 | SUPPORTS_VOD   | CLIENT_SETUP, SERVER_SETUP | {{negotiation}} |
| 0xVOD2 | MODE           | SUBSCRIBE, PUBLISH_OK, REQUEST_UPDATE | {{mode-param}} |
| 0xVOD3 | GROUP_INTERVAL | SUBSCRIBE, PUBLISH_OK, REQUEST_UPDATE | {{group-interval-param}} |
| 0xVOD6 | LARGEST_LOCATION | REQUEST_UPDATE | {{publisher-transition}} |
| 0xVOD7 | START_GROUP_OFFSET | SUBSCRIBE, PUBLISH_OK, REQUEST_UPDATE | {{start-group-offset-param}} |
| 0xVOD8 | MAX_SUBSCRIPTION_SUBGROUPS | SUBSCRIBE, PUBLISH_OK, REQUEST_UPDATE | {{max-sub-subgroups-param}} |

## MOQT Control Messages Registry

This document registers the following entries in the "MOQT Control
Messages" registry:

| Value  | Message Name | Specification |
|--------|--------------|---------------|
| 0xVOD9 | MAX_SUBSCRIPTION_SUBGROUPS | {{max-sub-subgroups-msg}} |

## MOQT Extension Headers Registry

This document registers the following entries in the "MOQT Extension
Headers" registry:

| Value  | Name | Scope | Specification |
|--------|------|-------|---------------|
| 0xVOD3 | GROUP_INTERVAL | Track | {{group-interval-ext}} |
| 0xVOD4 | LIVE_EDGE_DELTA | Object | {{live-edge-delta}} |

Note: GROUP_INTERVAL uses the same type value as both a SUBSCRIBE
parameter and a Track Extension Header.

--- back

# Acknowledgments
{:numbered="false"}

Claude (Anthropic) assisted with drafting and refining this document.

# Changelog
{:numbered="false"}

## draft-frindell-moq-vod-live-00
{:numbered="false"}

- Initial version
