# FRER Interactions Details

## Stream Filter Instance

PSFP is the IEEE 802.1Qci mechanism that applies per-stream checks such as stream gates, flow meters, and maximum Service Data Unit size filtering. AUTOSAR’s Ethernet Switch Driver specification also describes stream filtering as the selection process that determines the stream gate, flow meter, maximum SDU filtering, and internal priority handling for a received frame.

### 1. What a stream filter instance is

A **stream filter instance** is the table entry that says:

“For frames matching this stream handle and priority condition, apply this gate, this meter, this maximum SDU rule, and these counters/actions.”

Conceptually:

Frame arrives

↓

Ingress parser

↓

Stream identification assigns stream_handle

↓

Stream filter instance is selected

↓

Apply:

- stream gate

- flow meter

- maximum SDU check

- counters

- blocked-state logic

↓

Forward, drop, mark, or block

So the stream filter instance is not the first classifier. It normally uses a **stream handle** that was already assigned by stream identification.

### 2. Meaning of each field

| **Field**                            | **Meaning**                                                                                                                                                           |
|--------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Filter ID**                        | Identifier of this stream filter table entry.                                                                                                                         |
| **stream_handle spec**               | Says which stream handle this filter applies to. It may be exact or wildcard-like depending on implementation/configuration.                                          |
| **priority spec**                    | Says which priority values match. It may be a specific priority, for example priority 5, or wildcard priority, meaning any priority.                                  |
| **gate ID**                          | Points to the stream gate to use. The gate decides whether the frame is allowed by time window, open/closed state, optional IPV, and optional interval octet maximum. |
| **meter ID**                         | Points to the flow meter to use. The meter checks bandwidth behavior using CIR, CBS, EIR, EBS, color mode, and drop rules.                                            |
| **maximum SDU size**                 | Maximum allowed Service Data Unit size for frames matching this filter. Oversized frames are dropped.                                                                 |
| **counters**                         | Counts passed frames, dropped frames, oversized frames, meter drops, gate drops, etc., depending on implementation.                                                   |
| **blocked-due-to-oversize controls** | Optional behavior where an oversize violation can latch the stream into a blocked state.                                                                              |

The IEEE 802.1Qcr draft material describes PSFP managed objects in terms of stream filters, stream gates, and stream filter specifications, which is the same conceptual structure: filters select the treatment, while gates and meters perform the actual gate/meter behavior. ([IEEE 802](https://1.ieee802.org/wp-content/uploads/2019/03/802-1Qcr-D0-5.pdf?utm_source=chatgpt.com))

### 3. Simple example

Assume the switch has already identified an incoming AVTP Control Format stream and assigned it a stream handle:

stream_handle = ACF_STEERING_STATUS_A

The stream filter table may contain:

Filter ID: 17

stream_handle: ACF_STEERING_STATUS_A

priority spec: wildcard

gate ID: Gate_5

meter ID: Meter_5

maximum SDU: 256 bytes

This means:

Any frame with stream_handle ACF_STEERING_STATUS_A

regardless of received priority

must pass Gate_5,

must be checked by Meter_5,

must not exceed 256 bytes.

So the stream filter instance is the point where the stream’s **policy package** is attached.

### 4. Now connect this to FRER

With **Frame Replication and Elimination for Reliability**, or **FRER**, one original stream is replicated into multiple member streams.

Original stream S

↓

FRER replication

↓

Member stream S-A over Path A

Member stream S-B over Path B

Ideally, the switch should treat the member streams intentionally.

For example:

S-A → stream_handle = ACF_STEERING_STATUS_A → Filter 17 → Meter_A

S-B → stream_handle = ACF_STEERING_STATUS_B → Filter 18 → Meter_B

or, if the design intentionally meters the recovered compound stream later:

S-A and S-B pass toward FRER recovery

FRER eliminates duplicate

Recovered stream is metered once

The dangerous case is accidental grouping.

### 5. What “too broad wildcarding” means

Suppose the stream filter is configured too broadly:

Filter ID: 30

stream_handle: ANY

priority spec: wildcard

gate ID: Gate_ACF

meter ID: Meter_ACF

maximum SDU: 256 bytes

This may accidentally match both FRER member streams:

ACF_STEERING_STATUS_A → matches Filter 30

ACF_STEERING_STATUS_B → matches Filter 30

Now both redundant copies are sent through the **same gate and same meter**.

Copy A, sequence 100 → Filter 30 → Meter_ACF

Copy B, sequence 100 → Filter 30 → Meter_ACF

The flow meter sees two frames, even though they are duplicates of the same original packet.

### 6. Why this is a problem

Assume the original stream rate is:

100 packets per second

After FRER replication:

Member stream A = 100 packets per second

Member stream B = 100 packets per second

If both are accidentally matched to the same filter and same meter before duplicate elimination, the meter sees:

200 packets per second

If the meter was configured for only the original 100 packets per second, then the replicated traffic may be wrongly classified as excessive.

Result:

Copy A consumes meter tokens

Copy B consumes more meter tokens

Later frames become yellow/red

Red frames are dropped

Yellow frames may be dropped if DropOnYellow is enabled

So a perfectly valid FRER-protected stream can be damaged by a bad filter configuration.

### 7. Incorrect stream identification case

The same problem can happen even without wildcarding if stream identification itself is wrong.

Example intended mapping:

Path A copy → stream_handle = STEERING_A

Path B copy → stream_handle = STEERING_B

Incorrect mapping:

Path A copy → stream_handle = STEERING

Path B copy → stream_handle = STEERING

Now both member streams look identical to the stream filter stage:

STEERING → Filter 10 → Meter_10

STEERING → Filter 10 → Meter_10

The stream filter cannot know that this grouping was accidental. It simply applies the configured policy.

### 8. Gate-related accident

The issue is not only metering. A too-broad filter can also attach the wrong stream gate.

Suppose Path A and Path B have different expected arrival windows:

Member stream A expected window: 100 µs to 130 µs

Member stream B expected window: 140 µs to 170 µs

Correct configuration:

S-A → Gate_A

S-B → Gate_B

Incorrect wildcard configuration:

S-A → Gate_A

S-B → Gate_A

Now member stream B may arrive correctly for its own path, but incorrectly for Gate_A:

S-B arrives at 150 µs

Gate_A closed at 150 µs

S-B dropped

This is a false timing violation created by wrong stream filter association.

### 9. Maximum SDU-related accident

The same applies to maximum SDU size.

Suppose two different member streams or stream classes have different expected sizes:

Control ACF stream: max SDU = 128 bytes

Diagnostic ACF stream: max SDU = 512 bytes

A broad wildcard filter may accidentally apply the 128-byte rule to both:

ACF_CONTROL_\* and ACF_DIAG_\* → same filter → max SDU 128 bytes

Then a valid 300-byte diagnostic frame gets dropped as oversized, even though it is valid for its own stream contract.

### 10. Summary

A **stream filter instance** selects the per-stream treatment for a frame after stream identification.

It matches using a stream-handle specification and priority specification, then applies the configured stream gate, flow meter, maximum SDU check, counters, and optional blocked-state behavior.

In an FRER design, this configuration must be precise.

If stream identification is wrong, or if wildcard matching is too broad, multiple FRER member streams may accidentally share the same filter, gate, or meter.

Then duplicate copies can consume the same meter budget, be tested against the wrong gate window, or be dropped by an inappropriate maximum SDU rule before FRER recovery has a chance to eliminate duplicates.

## Maximum SDU Filter

**Maximum SDU filter** means:

For this identified stream, the switch is told: “Do not allow frames whose service data unit is larger than this configured size.”

Here **SDU** means **Service Data Unit**. In simple terms, it is the useful payload portion being checked for that stream, not the whole physical Ethernet frame including every link-layer overhead byte.

### 1. What the filter does

Suppose a stream is configured like this:

Stream: ACF_BRAKE_STATUS

Maximum SDU size: 128 bytes

Action: Drop if SDU > 128 bytes

Then:

Frame with SDU = 80 bytes → pass

Frame with SDU = 120 bytes → pass

Frame with SDU = 180 bytes → drop

This is useful because many deterministic streams should have a known maximum size. For example, a small control-status stream should not suddenly send a 900-byte payload.

### 2. Where the confusion comes from

You wrote:

FRER cannot recover a talker-wide oversize fault because both redundant copies contain the same oversize frame.

This means the fault is created **before** Frame Replication and Elimination for Reliability, or **FRER**, has a chance to help.

FRER works like this:

Talker sends one valid frame

↓

FRER sequence generation / replication

↓

Copy A goes on path A

Copy B goes on path B

↓

Listener / recovery function accepts first good copy

and discards the duplicate

FRER protects against problems such as:

Path A loses the frame

Path B still delivers it

→ FRER recovers

But it cannot protect against:

Talker creates an invalid oversized frame

Both replicated copies are oversized

Both copies violate the Maximum SDU filter

Both copies are dropped

→ no valid copy remains

### 3. Concrete example

Assume this stream contract:

Stream name: ACF_CAN_TUNNEL_01

Expected maximum SDU: 256 bytes

FRER replication: enabled

Path A: Switch 1 → Switch 3

Path B: Switch 2 → Switch 4

Normal case:

Talker creates SDU = 180 bytes

copy A = 180 bytes → passes Maximum SDU filter

Talker

copy B = 180 bytes → passes Maximum SDU filter

FRER recovery accepts one copy and discards the duplicate.

Oversize fault case:

Talker bug creates SDU = 400 bytes

copy A = 400 bytes → dropped by Maximum SDU filter

Talker

copy B = 400 bytes → dropped by Maximum SDU filter

FRER recovery receives no acceptable copy.

So the issue is not that one redundant path failed. The issue is that the **source produced a bad frame**, and FRER faithfully replicated that bad frame.

### 4. Why this is called a talker-wide fault

A **talker-wide fault** means the error is already present at the Talker output.

Bad frame generated once

↓

FRER creates two copies of the same bad frame

↓

Both copies are bad

This is different from a path-specific fault:

Good frame generated once

↓

FRER creates two good copies

↓

Path A corrupts/drops one copy

Path B delivers one good copy

↓

FRER succeeds

FRER is excellent for **path diversity faults**. It is not a cure for **common-mode faults**, where every replicated copy has the same defect.

### 5. What “optionally latch a stream-blocked state” means

Some configurations do more than simply drop the oversized frame.

They may say:

If this stream ever sends an oversized SDU,

mark the stream as blocked.

Then the switch behavior becomes:

First oversized frame arrives

↓

Drop that frame

↓

Set stream-blocked state

↓

Drop future frames from the same stream,

even if later frames are normal,

until management/reset clears the blocked state

This is useful for safety/security because a stream that violates its size contract may indicate:

faulty Talker software

wrong configuration

malicious injection

wrong encapsulation

broken gateway

### 6. Why FRER cannot fix this

FRER does not:

shrink oversized frames

fragment oversized frames

repair invalid payload sizes

generate a replacement frame

override stream policing

FRER only says:

If multiple valid redundant copies arrive,

accept one and eliminate duplicates.

But in the oversize case:

copy A is invalid

copy B is invalid

So there is no valid copy for FRER to select.

### 7. Summary

**The Maximum SDU filter enforces the stream’s allowed packet size. If the Talker itself creates an oversized frame, FRER replicates the same oversized frame onto all redundant paths. Since every copy violates the same size rule, every copy is dropped, so FRER has nothing left to recover.**

## Stream Gate

A **stream gate** can independently block or pass each replicated FRER member stream. If one redundant copy arrives at a switch at the wrong time, that copy may be dropped by its stream gate, but the other redundant copy on the other path may still arrive inside its valid gate window and reach the FRER recovery point.

### 1. What a stream gate is

A **stream gate** is a per-stream decision point in **Per-Stream Filtering and Policing**, often called **PSFP** or **IEEE 802.1Qci**.

After a frame is identified as belonging to a stream, the stream filter can point to a stream gate. The gate can then decide whether the frame is allowed to continue.

Conceptually:

Ingress frame

↓

Stream identification

↓

Stream filter

↓

Stream gate

↓

Pass or drop

A stream gate can have:

| **Stream gate property**            | **Meaning**                                                                                                                                                                                   |
|-------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Open / closed state**             | If open, matching frames can pass. If closed, matching frames are blocked.                                                                                                                    |
| **IPV**                             | Internal Priority Value used for traffic-class selection. AUTOSAR describes EthSwtStreamGateIPV as the internal priority value that determines the assigned traffic class, with range 0 to 7. |
| **Optional control list**           | A time schedule saying when the gate is open or closed.                                                                                                                                       |
| **Optional interval octet maximum** | A byte budget for a gate interval. Even if the gate is open, too many bytes in that interval can cause later frames to be blocked or dropped.                                                 |

AUTOSAR’s switch model shows the same processing idea: PSFP is an optional frame-processing stage, and a stream gate table contains stream gate configurations referenced by stream filter entries. ([autosar.org](https://www.autosar.org/fileadmin/standards/R23-11/CP/AUTOSAR_CP_SWS_EthernetSwitchDriver.pdf))

### 2. Simple example without FRER

Assume a stream gate is configured like this:

Stream: ACF_BRAKE_STATUS

Gate open window: 100 µs to 140 µs in every cycle

Gate closed outside that window

Now frames arrive:

Frame arrives at 110 µs → gate open → pass

Frame arrives at 155 µs → gate closed → drop

This is a **timing mismatch**. The frame may be semantically correct, its sequence number may be correct, and its size may be correct. But it arrived outside the gate’s allowed time window.

### 3. Now add FRER

With **Frame Replication and Elimination for Reliability**, or **FRER**, one original stream is replicated into multiple **member streams** sent over different paths. IEEE describes 802.1CB as supporting identification and replication of frames for redundant transmission, identification of duplicates, and elimination of duplicates. ([IEEE 802](https://1.ieee802.org/tsn/802-1cb/?utm_source=chatgpt.com))

Example:

Original stream S

↓

FRER replication

↓

Member stream S-A → Path A

Member stream S-B → Path B

At the receiver or recovery point:

If S-A arrives first and is valid → accept S-A, discard later duplicate S-B

If S-A is lost/dropped but S-B arrives → accept S-B

If both are dropped → recovery fails

If one member stream may be blocked by a timing mismatch while the other member stream still recovers the packet.

Original packet sequence number = 51

FRER creates two copies:

Copy A: member stream S-A, path A

Copy B: member stream S-B, path B

Now each path has its own stream gate behavior.

Path A gate window: 100 µs to 140 µs

Path B gate window: 130 µs to 170 µs

Actual arrival:

Copy A arrives at 150 µs

→ Path A stream gate is closed

→ Copy A is dropped

Copy B arrives at 145 µs

→ Path B stream gate is open

→ Copy B passes

→ FRER recovery receives sequence number 51

→ Packet is recovered

One redundant copy was blocked because it arrived outside its configured gate window.

The other redundant copy still passed through its own gate.

Therefore FRER still had one valid copy to deliver.

### 5. Why this is different from the Maximum SDU case

This is the key distinction.

### Maximum SDU oversize fault

Talker creates bad oversized packet

↓

FRER replicates bad packet

↓

Copy A oversized → dropped

Copy B oversized → dropped

↓

No recovery

That is a **common-mode fault**. Both copies are bad before path separation.

### Stream gate timing mismatch

Talker creates valid packet

↓

FRER replicates valid packet

↓

Copy A arrives at wrong gate time → dropped

Copy B arrives at correct gate time → passes

↓

FRER recovery succeeds

That is a **path-specific timing fault**. One redundant path failed the timing rule, but the other path did not.

### 6. Interval octet maximum case

A stream gate may also have an **interval octet maximum**, meaning a byte budget for a time interval.

Example:

Gate interval: 100 µs to 200 µs

Interval octet maximum: 500 bytes

Frames in that interval:

Frame 1 = 200 bytes → pass, used budget = 200

Frame 2 = 200 bytes → pass, used budget = 400

Frame 3 = 200 bytes → exceeds 500-byte budget → drop

In FRER:

Path A gate budget already consumed

Copy A exceeds interval octet maximum → dropped

Path B gate budget still available

Copy B passes → FRER recovers

So the same idea applies: **one member stream can be blocked by its local gate condition, while the other member stream survives.**

### 7. Summary

A **stream gate** is a per-stream pass/drop control point. It may block frames because the gate is closed, because the frame arrived outside its scheduled open window, or because the interval byte budget has been exceeded. In an FRER design, each redundant copy is carried as a separate member stream on a separate path. Therefore, one member stream can be dropped by a stream gate on one path, while the other member stream passes through a different gate on another path. FRER can then recover the packet from the surviving copy.

## IPV Result

The **IPV result** is the priority value produced by the stream gate. It can either be **null**, meaning “do not override priority,” or a configured **internal priority value**, meaning “use this priority inside the switch for traffic-class and queue selection.”

Here **IPV** means **Internal Priority Value**.

### 1. What “null or internal priority value” means

A stream gate can produce one of two outcomes:

IPV result = null

or:

IPV result = 0, 1, 2, 3, 4, 5, 6, or 7

If the IPV result is **null**, the switch continues using the frame’s normal priority. That normal priority may have come from the received VLAN **PCP**, or from ingress default priority, or from priority regeneration.

If the IPV result is a number, the switch uses that number as the **internal priority** for traffic-class selection.

### 2. What “overrides received priority” means

Suppose an incoming AVTP/ACF frame arrives with:

Received VLAN PCP = 2

Normally the switch may do:

PCP 2 → internal priority 2 → Traffic Class 2 → Queue 2

But if the stream gate says:

IPV result = 5

then the switch does:

Received PCP 2

↓

Stream filter matches protected stream

↓

Stream gate gives IPV = 5

↓

Use internal priority 5 for traffic-class selection

↓

Traffic Class 5 → Queue 5

So the frame came in with **PCP 2**, but inside the switch it is treated as priority **5** for selecting the queue.

### 3. “For traffic-class selection only” is very important

This does **not automatically mean** the switch rewrites the outgoing VLAN PCP field.

It means:

Use IPV to choose internal traffic class / queue / scheduler treatment.

It does not necessarily mean:

Change the VLAN PCP bits in the transmitted Ethernet frame.

Those are separate operations.

| **Operation**               | **Meaning**                                                                               |
|-----------------------------|-------------------------------------------------------------------------------------------|
| **IPV override**            | Internal decision: which traffic class or queue should this frame use?                    |
| **PCP rewrite / remarking** | External on-wire decision: what PCP value should be transmitted in the outgoing VLAN tag? |

So this is possible:

Ingress PCP = 2

IPV = 5

Internal queue = TC5

Egress PCP still = 2

Or, if the switch is separately configured to remark PCP:

Ingress PCP = 2

IPV = 5

Internal queue = TC5

Egress PCP rewritten to 5

But the IPV itself is about **internal traffic-class selection**.

### 4. Why this is useful for protected streams

Now consider a protected FRER stream.

A Talker sends the same packet through two redundant paths:

Original stream

↓

FRER replication

↓

Member stream A over Path A

Member stream B over Path B

Because of VLAN translation, ingress port defaults, or PCP rewriting, the two copies may arrive with different PCP values:

Copy A arrives with PCP 3

Copy B arrives with PCP 5

Without IPV override, the switch may place them into different queues:

Copy A → PCP 3 → TC3

Copy B → PCP 5 → TC5

That can create unequal delay, jitter, or even cause one copy to miss a gate window.

With IPV configured:

Stream gate IPV = 5

both copies can be forced into the intended internal traffic class:

Copy A: PCP 3 → IPV 5 → TC5

Copy B: PCP 5 → IPV 5 → TC5

This keeps the protected stream’s internal treatment consistent even if the received PCP is not consistent.

### 5. Example with ACF and FRER

Assume:

Stream: ACF_STEERING_STATUS

Expected queue: TC5

FRER: enabled

Path A copy arrives:

VLAN PCP = 3

Stream ID = ACF_STEERING_STATUS_A

Path B copy arrives:

VLAN PCP = 4

Stream ID = ACF_STEERING_STATUS_B

The switch stream gate is configured:

For both member streams:

IPV result = 5

Then the switch behavior is:

Copy A:

PCP 3 received

stream gate IPV = 5

TC5 selected

Copy B:

PCP 4 received

stream gate IPV = 5

TC5 selected

So even though the two copies arrived with different PCP values, both receive the same internal queue treatment.

### 6. Why this matters with PCP rewriting

PCP may be changed at network boundaries. For example:

ECU sends PCP 5

Gateway rewrites PCP to 3

Switch ingress default maps untagged frame to priority 2

Another bridge regenerates priority differently

If your safety or time-sensitive stream depends only on received PCP, this is fragile.

IPV allows the stream filter/gate to say:

I have already identified this as the protected stream.

Regardless of the received PCP, put it in the intended internal queue.

This is why the line says:

Useful when a protected stream must be placed in a specific queue regardless of PCP rewriting.

### 7. Summary

**The IPV result is the stream gate’s optional internal-priority override. If it is null, the switch uses the frame’s normal regenerated priority. If it is a configured value, the switch uses that value to select the traffic class and queue. This is useful for FRER/TSN protected streams because the stream can be classified into the intended queue even if the received VLAN PCP has been changed, removed, or regenerated differently along the path.**

## Flow Meter

A **flow meter** checks whether a stream is using more bandwidth than allowed. If **Frame Replication and Elimination for Reliability**, or **FRER**, duplicate copies are metered **before duplicate elimination**, both copies can consume the meter’s token budget even though only one copy will finally be delivered.

### 1. What the flow meter does

A **flow meter** is part of **Per-Stream Filtering and Policing**, also called **PSFP** or **IEEE 802.1Qci**.

It checks whether a stream is behaving within its configured bandwidth profile.

Conceptually:

Frame arrives

↓

Stream identification

↓

Stream filter

↓

Stream gate

↓

Flow meter

↓

Pass, mark, or drop

The flow meter is usually based on a token-bucket style model.

### 2. Meaning of CIR, CBS, EIR, EBS

| **Parameter** | **Expansion**              | **Meaning**                                 |
|---------------|----------------------------|---------------------------------------------|
| **CIR**       | Committed Information Rate | The guaranteed/expected rate for the stream |
| **CBS**       | Committed Burst Size       | How much committed burst traffic is allowed |
| **EIR**       | Excess Information Rate    | Extra rate allowed above committed rate     |
| **EBS**       | Excess Burst Size          | How much excess burst traffic is allowed    |

A simplified two-bucket view is:

Committed bucket:

filled at CIR

maximum size = CBS

Excess bucket:

filled at EIR

maximum size = EBS

When a frame arrives, the meter checks whether enough tokens exist.

Enough committed tokens → green frame

No committed tokens,

but enough excess tokens → yellow frame

Not enough tokens → red frame

Then the configuration decides what to do.

green → pass

yellow → pass or drop, depending on DropOnYellow

red → drop

### 3. What the controls mean

### CIR and CBS

These define the normal allowed bandwidth.

Example:

CIR = 10 Mb/s

CBS = 1500 bytes

This means the stream is expected to behave like a 10 Mb/s stream, with a limited burst allowance.

### EIR and EBS

These define additional excess bandwidth.

Example:

EIR = 2 Mb/s

EBS = 500 bytes

This means the stream may temporarily exceed the committed rate, but only within the excess allowance.

### Coupling flag

The **coupling flag** controls interaction between the committed bucket and the excess bucket.

A practical way to understand it:

If the committed bucket is already full,

can unused committed-rate token generation help the excess bucket?

Depending on the configured coupling behavior, unused committed capacity may or may not contribute to excess allowance.

### Color mode

The meter can be:

color-blind

or:

color-aware

In **color-blind** mode, the meter ignores any previous color/classification and meters the frame afresh.

In **color-aware** mode, the frame may already carry an internal color state such as green, yellow, or red from an earlier policing stage, and the meter takes that into account.

### DropOnYellow

This controls whether yellow frames are allowed.

DropOnYellow = false

green passes

yellow passes

red drops

DropOnYellow = true

green passes

yellow drops

red drops

So **DropOnYellow** makes the meter stricter.

### MarkAllFramesRed

This is a control that forces all frames to be treated as red.

In effect:

MarkAllFramesRed enabled

↓

all matching frames become red

↓

red frames are discarded

This can be used for blocking, diagnostics, fault containment, or management-controlled shutdown of a stream.

### 4. Now connect this to FRER

With **FRER**, the same original packet is replicated.

Original packet, sequence number 100

↓

FRER replication

↓

Copy A, sequence number 100 → Path A

Copy B, sequence number 100 → Path B

At the recovery point, FRER should accept one copy and discard the duplicate.

Copy A arrives first → accept

Copy B arrives later → duplicate, discard

That is normal FRER behavior.

### 5. The problem: meter before recovery

Now assume the flow meter is placed **before** the FRER recovery function.

Copy A arrives

↓

Flow meter

↓

FRER recovery

Copy B arrives

↓

Flow meter

↓

FRER recovery

The meter does not yet know that Copy B will later be eliminated as a duplicate. To the meter, Copy B is just another frame consuming bandwidth.

So both copies may consume meter budget.

Copy A consumes tokens

Copy B also consumes tokens

FRER later discards Copy B as duplicate

**Duplicate copies may consume meter budget before duplicate elimination if the meter is before recovery.**

### 6. Concrete example

Assume:

Original stream rate: 100 frames per second

FRER replication: 2 copies per original frame

Meter placed before FRER recovery

Meter budget: sized for 100 frames per second

The Talker sends:

Packet 1 → replicated into Copy A1 and Copy B1

Packet 2 → replicated into Copy A2 and Copy B2

Packet 3 → replicated into Copy A3 and Copy B3

The meter sees:

A1, B1, A2, B2, A3, B3 ...

So the meter sees **200 frames per second**, even though the application intended only **100 original packets per second**.

Result:

Meter budget is consumed twice as fast

Some valid copies may become yellow or red

Red copies are dropped

Yellow copies may also be dropped if DropOnYellow is enabled

This can cause false policing.

### 7. Why this matters

Suppose the meter has enough tokens for only one 500-byte packet in the current interval.

Available committed tokens = 500 bytes

Packet size = 500 bytes

FRER copies arrive:

Copy A = 500 bytes

Copy B = 500 bytes

Meter-before-recovery behavior:

Copy A:

consumes 500 tokens

marked green

passes

Copy B:

no committed tokens left

may become yellow or red

may be dropped

FRER recovery:

accepts Copy A

would have discarded Copy B anyway

Here, the immediate packet still survives.

But now consider the next original packet:

Copy A of next packet arrives

tokens are still depleted

frame may become yellow or red

may be dropped

So the duplicate copy wasted meter budget and can indirectly harm later non-duplicate frames.

### 8. Better ordering: recovery before meter

If the design allows FRER recovery before metering, the pipeline becomes:

Copy A arrives

↓

FRER recovery accepts sequence 100

↓

Flow meter meters one delivered packet

Copy B arrives

↓

FRER recovery detects duplicate sequence 100

↓

Duplicate discarded before flow meter

Now the meter only sees:

one frame per original packet

That is usually the cleaner behavior if the meter is intended to police the **compound recovered stream**, not the individual member streams.

### 9. But sometimes metering before recovery is intentional

Metering before recovery is not automatically wrong.

It may be intentional if the designer wants to police each **member stream** separately.

Example:

Member stream A on Path A has its own meter

Member stream B on Path B has its own meter

Then each path is checked independently:

Path A meter checks whether member stream A is behaving

Path B meter checks whether member stream B is behaving

This can be useful when each redundant path has its own bandwidth contract.

The danger appears when duplicated traffic is metered as if every copy were an independent original packet, especially when the meter budget was configured for the original stream rate rather than the replicated rate.

### 10. Summary

A **flow meter** enforces the bandwidth contract of a stream using committed and excess token-bucket parameters. Frames are classified as green, yellow, or red. Red frames are discarded, and yellow frames may also be discarded depending on configuration. In an FRER system, if the flow meter is located before the FRER recovery function, both redundant copies of the same original packet can consume meter tokens. The later duplicate may be eliminated by FRER, but by then it has already consumed meter budget. This can cause unnecessary yellow/red marking or drops unless the meter is sized for replicated traffic, applied separately per member stream, or placed after duplicate elimination.
