=================
The Tao of DelTiC
=================

(c) 2024 Jonathan Morton


Abstract
--------

DelTiC (Delay Time Control) is a modern time-domain AQM based on a delta-sigma control loop, derived from PID control theory.  This document explains the how and why of DelTiC's operation, and other algorithms incorporated into the family of qdiscs built around DelTiC itself:  POLYA, ZERNO, BOROSHNE, and KHLIB.

On the transport side, the MASLO overlay pacer and the ESSP slow-start algorithm are described in a separate, companion document.


Background
----------

The need for Internet congestion control was established by the ARPAnet congestion-collapse events of the mid-1980s.  The initial solutions adopted were: for connectionless protocols, an exponential backoff schedule for retries; and for reliable stream protocols (principally TCP), the two-phase method of exponential slow-start and AIMD congestion-avoidance, standardised as the Reno algorithm.  The only reliable congestion signal was the observance of lost packets due to the packet queue overflowing at the bottleneck link.  For unreliable datagram protocols that maintained a persistent connection, applications were expected to either implement a Reno-compatible scheme, or ensure that their traffic was non-saturating.

In those days, memory was sufficiently expensive that packet buffers were necessarily small, and these congestion indications tended to appear almost as soon as the link was saturated.  In the following decades, memory has grown cheaper and more abundant faster than link speeds have increased, so in some cases many seconds of traffic can be buffered before losses occur due to overflow.  This has significant effects on reliability, as connectionless protocols (eg. DNS) and the establishment of new connections cannot operate if the delay due to buffering exceeds the protocol timeout.

One solution is to "right-size" the buffers, based on the observation that a buffer equal in capacity to the baseline BDP of the path is sufficient for full throughput with AIMD congestion control.  Analysis also shows that when there is a high degree of statistical multiplexing (as often occurs in backhaul and core networks), a smaller buffer also suffices; the guideline is BDP/√flows.  However, this solution is imperfect, not least due to the wide range of path RTTs that may be observed by flows passing through a single bottleneck.  It also still relies on packet loss as a congestion signal, which conflicts with ISPs' desire to provide a service which minimises such losses, which may be perceived as "unreliability".

ECN (Explicit Congestion Notification), standardised in RFC-3168, addresses this problem by providing a mechanism to signal congestion without dropping packets.  AQM (Active Queue Management) may then be implemented to apply ECN marks to some proportion of packets which signal ECN capability, and drop the same proportion of packets that are not ECN capable, when queuing exceeds some threshold but well before the queue overflows.  Although many queues in today's Internet still rely on the old "drop at tail" paradigm, AQM is increasingly widely deployed and has proved to be a useful mechanism.

Early AQMs such as RED were very crude mechanisms that were difficult to tune for correct operation.  Though improved algorithms appeared over time, there was a persistent debate over whether AQM was even the right mechanism for the job, or whether FQ (Fair Queuing) of a right-sized buffer was more suitable.

This debate was largely resolved by the introduction of CoDel (Controlled Delay) AQM, which introduced several novel approaches to AQM implementation, and successfully solved the tuning problem by moving all important parameters into the time domain.  A CoDel AQM could thus be deployed in the Internet with default settings in almost all cases.  Further, most deployments were of the FQ-CoDel variant which applied an individual CoDel AQM to each of the subqueues managed by a sophisticated FQ implementation, or of the even more sophisticated CAKE (Common Applications Kept Enhanced) which included a built-in shaper.  The question "FQ __or__ AQM" was answered "FQ __and__ AQM".

The other type of AQM that has risen to prominence in recent years is the PI family, which controls a probabilistic congestion signal (itself still similar to that used in RED) using a proportional-integral controller (from the PID family of algorithms).  These are usually pre-tuned by the implementing vendor for the expected range of network conditions, and can show impaired performance (eg. oscillation) when applied to links outside that range.

Though largely successful in practice, CoDel has been criticised for its control law, which has the virtue of simplicity (and thus ease of analysis), but fails to react quickly to flow startup transients or floods of uncontrolled traffic.  CAKE includes a probabilistic-drop mechanism based on the BLUE algorithm to mitigate the latter.

DelTiC was conceived to address CoDel's shortcomings while incorporating its more successful features.  Like CoDel, it operates in the time-domain and thus inherits CoDel's ease of tuning, including sensible defaults which are suitable for Internet traffic.  Marking and dropping is likewise done at the head of the queue, minimising the time lag between the signal being emitted and the response of the sender.  The main difference is in the control law, which is more similar to PI-family AQMs.

Another recent development is the research effort into practical alternatives to AIMD congestion control.  AIMD is characterised by a sawtooth behaviour, which stubbornly resists efforts to realise maximum-throughput and minimum-delay performance simultaneously.  To solve this problem, one prominent HFCC (High Fidelity Congestion Control) effort explores an AIAD scheme, exemplified by DCTCP, while we have adopted an MIMD scheme in MASLO.

A key hurdle in this area is achieving compatibility in the Internet with existing AIMD traffic, without either class starving out the other, while achieving incremental deployability.  In practice, this means that HFCC traffic must be capable of adopting conventional AIMD behaviour when the network is not HFCC-aware.  In particular, it is vital that transport endpoints can reliably distinguish a HFCC-aware network from one that is not.  Unfortunately, not all HFCC experiments presently conform to this requirement.

The SCE (Some Congestion Experienced) experiment has RFC-3168 compatibility as a central tenet, and uses a different marking codepoint from conventional ECN so that transport endpoints can unambiguously distinguish SCE signals and thus network capability.  DelTiC is designed to work with SCE as well as conventional ECN, and can be used with both MASLO and SCE-based versions of DCTCP.  The DelTiC-family qdiscs respect a DSCP identifier, so that SCE marks are not applied to non-SCE-capable traffic, and bottlenecks can queue the two categories separately.


POLYA - the Fertile Fields
--------------------------

Each of the DelTiC-family qdiscs is given a name from the Ukrainian language, related to agriculture and, more specifically, its principal staple crops and foods.  The simplest of these qdiscs is POLYA, implementing the minimum functionality of a FIFO with AQM.  The name evokes the exceptionally fertile "black earth" of Ukraine, which was once the principal agricultural region of the Soviet Union and now exports to large portions of Africa.

POLYA serves two purposes: as a testbed for initial development and tuning of DelTiC itself, and as a "leaf qdisc" to use within custom QoS hierarchies.  Beyond the basic and universal functions of a FIFO, three instances of the DelTiC AQM are tuned to apply SCE marks, conventional ECN marks, and to unconditionally drop packets in overload situations.  There is also a "jitter estimator" to improve the AQM's behaviour when the output link has significant MAC grant delays.  Under this heading, the DelTiC AQM and the jitter estimator algorithms will be described in detail.

### DelTiC (Delay Time Control) AQM

The core of DelTiC is a control law, which is processed for every packet dequeued, and emits a binary decision as to whether to emit a congestion signal for that packet.  The precise form of that congestion signal may be an SCE mark, a CE mark, or dropping the packet.  POLYA has three instances of the DelTiC control law, each focused on one of these responses and with different parameterisations.

There is one configurable parameter to the control law: a resonance *frequency*, dimensioned in Hertz.  For implementation convenience, a *target* is derived as the reciprocal of the frequency, dimensioned in seconds (though typically written in milliseconds).  For example, the default resonance frequency for CE marking is 40Hz, resulting in a target of 25ms.

Four persistent state variables are kept:  an *accumulator* associated with the *sigma* term of the control law, a *history* assocated with the *delta* term, a *timestamp* marking the time the previous packet was dequeued, and an *oscillator* phase to schedule marking.

The control law takes as input a *sojourn* time, representing the delay the packet presently being dequeued has experienced through waiting in the queue.  The precise calculation of *sojourn* is detailed under the description of the "jitter estimator".  The other input is *now*, a timestamp representing the time of dequeuing.

An *interval* between packet dequeue events is immediately calculated by subtracting *timestamp* from *now*.  This is clamped to one second to avoid any possibility of overflow in subsequent calculations.  If this clamping occurred and the *sojourn* time is less than *target*, then both the *accumulator* and *interval* are forced to zero on the assumption that the queue has been empty and idle.

The *history* state is simply the *sojourn* time of the previous dequeued packet (initialised to zero).  The *delta* term of the control law is the *sojourn* time minus the *history*, and corresponds to the D term of a PID controller.  This may make a positive or negative contribution to the control law, depending on the trend of the queue occupancy.

The *sigma* term is the *sojourn* time minus the *target*, multiplied by the *interval*.  This corresponds to the I term in a PID controller.  This may make a positive or negative contribution to the control law, depending on whether the sojourn time is above or below the target.

The *accumulator* is then updated by adding the sum of *sigma* and *delta*, multiplied by the resonance *frequency*.  The *accumulator* is clamped to the non-negative domain; if this clamping occurs, the *oscillator* phase is also reset to zero.  This ensures that marking does not begin prematurely after the link is newly saturated.

At this point, the *history* and *timestamp* state variables are updated, and processing then continues with the marking oscillator.  Marking is suppressed if *sojourn* is less than half of *target*, and in this case oscillator processing is completely skipped.

In the marking oscillator, the control law *accumulator* is interpreted as a scaling factor for a Numerically Controlled Oscillator.  When *accumulator* is 1.0, the oscillator beats at the resonant *frequency*.  The control law may act to increase this frequency considerably above this figure, or to mark steadily at a slower rate, depending on queue behaviour.  Every "beat" of the oscillator, which occurs when the *oscillator* state variable wraps around from its maximum value to zero, produces one marking event.

This is accomplished by adding the product of the *accumulator*, *interval*, and resonance *frequency* to the *oscillator*.  The *oscillator* is then checked for overflow (ie. exceeding a phase of 1.0), which triggers a marking event and subtracting 1.0 from *oscillator* to nominally bring it back into range.

If the oscillator experiences a *double overflow*, ie. is still above 1.0, this implies that the marking frequency has exceeded the packet dequeue rate.  To avoid having the accumulator grow without bound, this condition causes the *accumulator* to have one-sixteenth of its value removed.  This will help the control law to recover after a major positive excursion.

The above construction is a *delta-sigma* control law, which in the PID family omits the P term.  Its behaviour is similar in some ways to a PI controller, but the *delta* term differs from a *proportional* term in that it acts to positively damp the accumulator (which would normally be the independent *integral* term) when the sojourn time is trending towards the target, and accelerate it when it is trending away.  A proportional term has a tendency to sharpen the control response unconditionally, inducing oscillation under some path conditions, which is avoided here.

Keen observers will notice that when several such control laws are ganged on a single queue, the *interval* calculation and the conditional resetting of *accumulator* are in common between them, and only one *timestamp* state variable is then required.  This may be of interest where performance is critical.  Otherwise, the logic is mostly straight-line calculations with no table lookups or divisions.

### Jitter Compensation

It is commonly assumed in AQM design that packets are dequeued at a smooth rate, which is high enough that the interval between packets is small compared to the queue target.  But this is not always the case, and there are some common cases where reality differs markedly.

The PIE AQM incorporates a specific measure to compensate for the characteristics of DOCSIS uplinks, in which bursts of packets are dequeued at intervals corresponding to a MAC grant cycle.  WiFi also exhibits similar characteristics, though with greater variability in the interval between bursts.  Where the link capacity is particularly low (notably on analogue modems or rural ADSL lines), the packet serialisation delay which dominates the dequeue interval may be greater than the desired sojourn time target, such that the queue appears perpetually congested even when not fully utilised.

DelTiC estimates the magnitude of this effect, termed *jitter* (though it might not originate from a true jitter effect), by taking an interval-weighted moving average of intervals, with a time constant of one second.  The operative *interval* in this case is the same *interval* between packet dequeues as used in the main control law, with one key exception:  intervals where the queue was empty are not counted.

Where packets are dequeued at a smooth rate, the jitter estimate will converge to the serialisation time of individual packets.  Where packets are dequeued in bursts, the interval between packets in the burst will be near zero, causing these intervals to contribute very little to the moving average.  The jitter estimate will thus converge to the much larger interval *between* bursts.

When a packet is enqueued, DelTiC records its *enqueue time* in the packet metadata.  At the same time, if the packet was added to an otherwise empty queue, the dequeue *timestamp* is reset to the current time, thus excluding any time when the queue was empty from the estimate.

Then, when the packet is dequeued, the jitter estimate is first updated using a version of the EWMA formula: *jitter* = *interval* * *interval* + *jitter* * (1.0s - *interval*).  The packet's sojourn time is then calculated as (*now* - *enqueue time*) - *jitter*, clamped to the non-negative domain.  This "forgives" any delay estimated to be due to jitter from the sojourn time used in the control law.

It is important to note that this jitter estimation and compensation occurs without requiring any a-priori estimate or assumption about the link characteristics.  This is an important advantage over previous techniques in this area, as it allows adaptation to links whose characteristics are unknown (eg. the highly time-variable characteristics of WiFi networks) or unpublished (eg. in commercial wireless operations such as cell networks and satellite links), and eliminates a likely source of error in configuration.

### Elements specific to POLYA

POLYA is a simple, work-conserving, single-FIFO qdisc.  Packets are timestamped when they are enqueued, and there is one jitter compensator corresponding to this queue and the three DelTiC instances.

The primary DelTiC instance applies CE marks according to RFC-3168, ie. it drops instead of marks if the target packet is marked Not-ECT.  It has a resonant frequency of 40Hz by default, ie. targets 25ms sojourn time.

The secondary instance applies SCE marks to ECT packets, provided the primary instance has not already triggered for that packet.  It has a resonant frequency of 200Hz by default, ie. targets 5ms sojourn time.

A third instance drops packets, regardless of their ECN field, and is intended to handle severe overloads for which ECN marks are insufficient.  It has a resonant frequence of 8Hz by default, ie. targets 125ms sojourn time.

The 5:1 ratio of resonant frequencies is intended to give some working headroom between the target sojourn time of each instance and the lowest possible marking threshold of the next higher instance.  The 25ms effective target for RFC-3168 marking should keep traffic adhering to conventional congestion control algorithms closer to full throughput than the 5ms default target of CoDel.
