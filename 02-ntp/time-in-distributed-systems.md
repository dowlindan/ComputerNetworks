# Time in Distributed Systems Investigation

## Section 1: Learning Process

**Documenting of AI-assisted learning journey.**

The AI tool that I used was ChatGPT's free version. My first three prompts were the following:
- "1) Explain why even with NTP we can't perfectly sync clocks across machines? 2) How do we determine event order despite this? 3) Imagine one server records an event with a timestamp: 2:00:00.100, and another records an event with a timestamp 2:00:00.050. Do we know which event happened first, why or why not? 4) Explain why timestamps does not mean guaranteed ordering."
- "What are Lamport clocks and how do they work? Specifically, mention how they order events without wall-clock time, and the counter mechanism. Explain what logical clocks can and cannot determine, and situations where you still might need wall-clock time."
-"Explain the CAP theorem with simple examples. Specifically include what it stands for, the definition of each word, the theorem states you can only guarantee _ of three properties. Explain what traditional bank databases like MySQL choose, DynamoDB, and DNS systems choose. Then explain how time sync relates to Cap. If you choose consistency, do you need tight clock sync? If you choose AP, are loose clocks acceptable?"

The concept that was most confusing initially was CAP, particularly what each letter meant in the context of a distributed system. The follow-up questions that helped included asking for specific concrete examples, such as DNS-propagation. This example made it click because it is something that I can relate to by things that I do. Getting old data is common but is a necessity for availability.


## Section 2: When Time Goes Wrong - A Real Failure

Incident resarch: 2012 Leap Second Bug


I chose the 2012 leap second bug because it had the most coverage. This incident was a series of outages and system failures that occured during the previous leap second insertions, on June 30, 2012. Major websites and services were affected, around half the internet, was affected. What went wrong specifically was a bug that was the root cause. Many systems were not designed to handle a 61-second minute. Behavior was only tracked up to 60 seconds, and not after. During a leap second, however, a special timestamp is inserted. This would cause the Linux termina's timekeeping code to misbehave by ending up in a loop, leading to 100% CPU usage and causing systems to hang or crash. Major services experienced crashes or disruptions for around a few hours. Some services included reddit, cloudflare DNS, linkedin, yelp, and java-based applications. As mentioned before around half the internet was affected. After learning about CAP and consistency, it was clear that a major component was all of these major systems prioritizing availability. These systems kept runnnig. Logical clocks might have helped a little bit, if some logic depended on event ordering instead of timestamps, because the bug was mostly due to counting seconds. This was fundamentally not about clock accuracy, because inaccuracy did not determine whether the edge case was handled. It was about coordination because time protocols are never expected to have more than 60 seconds.


---

## Section 3: Physical Time vs Logical Time


### Part A: Why Physical Clocks Fail

1. **The core problem:**
   - Even with NTP, why can't we perfectly synchronize clocks across machines?
   - What does this mean for determining event order?

2. **Simple scenario:**
   - Server A records event with timestamp: `2:00:00.100`
   - Server B records event with timestamp: `2:00:00.050`
   - Which event happened first?
   - Can we know for certain? Why or why not?


Even with NTP, network latency is variable, time between endpoints is not equal, clock drift occurs due to several reasons, and there is no instant correction. NTP gives error which is never 0. Some ways to determine event order despite this include logical or lamport clocks, vector clocks, or hybrid logical clocks, instead of wall-clock time. We do not know which event happened first in the situation above because server clocks may differ by more than 50ms, and the timestamps are local not global. We need more information. Overall, physical clocks cannot be perfectly synchronized, so timestamps cannot reliably order events. Instead, logical clocks are used to determine casuality-based ordering.

### Part B: Logical Clocks - The Alternative

Lamport clocks order events in a distributed system without wall-clock time. They use a counter mechanism to capture relationships between events. A Lamport clock is a logical clock represented by a single integer counter maintained by each process. It establishes consistent ordering of events based on casualty. If A influences B, A happened before B. Each process maintains a counter. Before every event, the counter increments. When sending a message, the current value is attached to the message. When receiving a message with a timestamp, the counter is set to MAX(counter, T) + 1. This ensures if A happeend before B, cnt(A) < Lcnt(B). These clocks rely on message causality instead of time. These clocks cannot determine if two events are concurrent, whether an event happened earlier in real time, or duration or time gaps. These clocks do not measure elapsed time.

You still need wall time for user-facing timestamps, timeouts, performance measurement, scheduling, and legal requirements. Lamport clocks use monotonically increasing counters and message timestamps to order events based on casuality. They guarantee relation between events, but are unable to detect whether two events happened at the same time or provide real-time information, for which wall-clock time is still needed.


---

## Section 4: CAP Theorem and Eventual Consistency

**The fundamental tradeoffs in distributed systems.**

### Part A: CAP Theorem

CAP stands for Consistency, Availability, and Partition tolerance. The CAP theorem states that in the presence of a network partition, a distributed system can guarantee at most two of three properties. Consistency is all nodes seeing the same data at the same time. After a write completes, every read returns the latest value. Availability is every request receiving a respnse. Partition tolerance is that the system continues operating despite network failures. To keep consistency, you must stop answering requests on one side. To keep availability, both sides answer requests, losing consistency. MySQL is an example of CP, DynamoDB and DNS choose AP.
Regarding time sync, if you choose CP, tight clock sync is often required for transaction ordering. Clock uncertainty directly threatens consistency guarantees. For AP, loose clocks are acceptable because no global ordering is required and conflicts are resolved later. AP systems prefer logical clocks instead of wall-clock precision.

### Part B: Eventual Consistency 

Eventual consistency is a model where if no new updates are made to a distributed system, all replicas will eventually converge to the same value. The key idea is that a system may return stale data temporarily, but given enough time and no further writes, all nodes agree. Eventual means convergence over time, and not immediately. Convergence depends on network delays, partitions, and other conditions. An example is DNS propagation. A DNS record is updated. Some servers still cache the old IP, over time, all caches update and eventually everyone sees the same record, an example of AP. Regarding the NTP client, the clock is not perfectl instantly and converges gradually. When you run it, the clock offset is estimated, adjustments are made slowly, but time is never jumped backward abruptly. In an eventually consistent system, you don't need perfect clocks which is the point. It does not require immediate agreement. Eventual consistency explicitly acknowledges network conditions.

## Section 5: Where Your NTP Client Fits

**Connecting everything to the actual implementation**

### Part A: What The Client Provides

1. **Measuring implementation accuracy:**
   
   
   ```
   Run 1: offset = -61.22 ms
   Run 2: offset = -216.05 ms  
   Run 3: offset = -298.45 ms
   
   Average offset: -190.91 ms
   ```
   
   Typical NTP accuracy over the Internet: 10-100 milliseconds

2. **Classifing the client:**

   The client provides eventual consistency, which is clocks converging over time. Clock offsets are corrected gradually over time and different machines may observe different times at any instant, but given stable conditions, clocks converage within a bounded error which is never zero.

### Part B: Understanding the Big Picture

**Synthesis questions - connect all the concepts:**

1. **Can your NTP client solve the logical ordering problem from Section 3?**

My NTP client cannot solve the logical ordering problem from above. It only bounds clock error and gradually converges, so the 50 ms difference could be due to clock skew rather than real event order. WIthout logical clocks, NTP timestamps alone cannot determine which happened first.

2. **What does your NTP client actually provide?**
   
   Pick the BEST answer and explain your choice:
   - [ ] A way to order all events in a distributed system
   - [ x ] A shared reference time for logs, certificates, and coordination
   - [ ] Perfect synchronization between all machines
   - [ ] A replacement for logical clocks like Lamport clocks
   
   **Your choice:** B, a shared reference
   
   **Why?** (2-3 sentences)

   An NTP client provides a best-effort, bounded-error reference to actual time that gradually converges across machine. It does not perfectly synchronize clocks, cannot order events, and does not replace logical clocks.

3. **Complete the picture:**
   
   - Physical clocks (NTP) are needed for: real-world timekeeping such as logs, visible timestamps to users, timeouts, certificates, and anything that requires coordination with external systems.
   - "Logical clocks (Lamport) are needed for: ordering events based on causality in a system, independent of wall-time.
   - In a real distributed system, you typically need: both physical clocks for real-time coordination and observability, and logical clocks for correct event ordering and reasoning about causality.


---

