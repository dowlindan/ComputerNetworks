# NTP Protocol Design Investigation

## Investigation 1: Topic 3: Network Byte Order / Endianness

### Implementation Context
[What you coded and what puzzled you - reference specific code]

What I implemented: the `ntp_to_net` function below:

```C
void ntp_to_net(ntp_packet_t* packet){
    uint32_t root_delay_nl = htonl(packet->root_delay);
    uint32_t root_dispersion_nl = htonl(packet->root_dispersion);
    uint32_t reference_id_nl = htonl(packet->reference_id);
    packet->root_delay = root_delay_nl;
    packet->root_dispersion = root_dispersion_nl;
    packet->reference_id = reference_id_nl;

    ntp_ts_to_net(&(packet->ref_time));
    ntp_ts_to_net(&(packet->orig_time));
    ntp_ts_to_net(&(packet->recv_time));
    ntp_ts_to_net(&(packet->xmit_time));
}
```
as well as the inverse function which behaved the same but in reverse.


What puzzled me was the following:
1. Why do net protocols use big endian?
2. Simply what the shorthand function name means, h to nl?
3. A refresher in endianness on general, what does it mean for how bytes are read when changing between the two (SysProg refresher)?

### Investigation Journey  
#### Step 1: Get an idea for why net protocols use big endian
Some notes from the slides:
- ISA for the processor specifies which endian
- Endian: How to interpret byte order for multi-byte values
- 0x12345678 with big endian is read intuitively as 0x12, 0x34, 0x56, 0x78 -> 0x1234, 0x5678
- In little endian it is read as the first step, but then 0x3412, 0x7856 -> 0x78563412
- Older machines standardized big endian, and most RFCs were created when machines were big endian
- For little endian, converting between formats does not require bit shifting, but for big-endian it does. Certain math operations are simplified.

Some notes from online
- StackOverflow: RFC1700 observed big endian was a common convention

#### Step 2: Learn what htonl/ntohl means and why it is a C builtin
- Google AI overview, and IBM says: host to network long and network to host long. A long is at least 32 bits.
- These functions are essential to handle byte order differences for networking code, which is why they exist in the arpa/inet.h header file.
- A quick google search shows that converting between the two formats manually is not computationally hard if needed for whatever reason.

#### Step 3: Compare and contrast how endianness affects how something is read.

- This was encapsulated in step 1, but another example from online shows:
- Big Endian: ABCD1234
- Little Endian: 3412CDAB

#### Step 4. Run the program normally

```
--- Request Packet ---
Leap Indicator: 3
Version: 4
Mode: 3
Stratum: 0
Poll: 6
Precision: -20
Reference ID: [0x00000000] NONE
Root Delay: 0
Root Dispersion: 0
Reference Time: 1899-12-31 19:00:00.000000 (Local Time)
Original Time (T1): 1899-12-31 19:00:00.000000 (Local Time)
Receive Time (T2): 1899-12-31 19:00:00.000000 (Local Time)
Transmit Time (T3): 2026-01-31 01:07:48.558765 (Local Time)
ntp_fraction: 2811940858

Received NTP response from pool.ntp.org!
--- Response Packet ---
Leap Indicator: 0
Version: 4
Mode: 4
Stratum: 2
Poll: 6
Precision: -30
Reference ID: [0xA9E58086] 169.229.128.134
Root Delay: 215
Root Dispersion: 399
Reference Time: 2026-01-31 01:03:32.943857 (Local Time)
Original Time (T1): 2026-01-31 01:07:48.558765 (Local Time)
Receive Time (T2): 2026-01-31 01:07:48.430354 (Local Time)
Transmit Time (T3): 2026-01-31 01:07:48.430398 (Local Time)

=== NTP Time Synchronization Results ===
Server: pool.ntp.org
Server Time: 2026-01-31 01:07:48.430398
Client Time: 2026-01-31 01:07:48.654705
Round Trip Delay: 0.095896

Time Offset: -0.176359
Final dispersion: 0.055677
Your clock is running BEHIND by 176.36ms
Your estimated time error will be +/- 55.68ms
```

#### Step 5. Run the program WITHOUT converting between endians (ntp_to_net and ntp_to_host fns commented out)

```
--- Request Packet ---
Leap Indicator: 3
Version: 4
Mode: 3
Stratum: 0
Poll: 6
Precision: -20
Reference ID: [0x00000000] NONE
Root Delay: 0
Root Dispersion: 0
Reference Time: 1899-12-31 19:00:00.000000 (Local Time)
Original Time (T1): 1899-12-31 19:00:00.000000 (Local Time)
Receive Time (T2): 1899-12-31 19:00:00.000000 (Local Time)
Transmit Time (T3): 2026-01-31 01:06:08.985139 (Local Time)
ntp_fraction: 146252226

Received NTP response from pool.ntp.org!
--- Response Packet ---
Leap Indicator: 0
Version: 4
Mode: 4
Stratum: 4
Poll: 6
Precision: -25
Reference ID: [0xE4067728] 228.6.119.40
Root Delay: 1275920384
Root Dispersion: 2214789120
Reference Time: 1973-06-02 02:25:49.256777 (Local Time)
Original Time (T1): 2026-01-31 01:06:08.985139 (Local Time)
Receive Time (T2): 1942-01-20 21:54:37.804008 (Local Time)
Transmit Time (T3): 1942-01-20 21:54:37.469337 (Local Time)

=== NTP Time Synchronization Results ===
Server: pool.ntp.org
Server Time: 1942-01-20 21:54:37.469337
Client Time: 2026-01-31 01:06:09.034051
Round Trip Delay: 0.383583

Time Offset: -2651713891.372923
Final dispersion: 43529.691791
Your clock is running AHEAD by 2651713891372.92ms
Your estimated time error will be +/- 43529691.79ms
```

#### Step 6. Contrast the results 

It's very clear how the results differ. Specifically, the reference time, receive time, and transmit time and therefore all subsequent calculations are off by extremely large amounts. The consequences of not switching between endians is apparent, as data being improperly interpreted is terrible among all applications. This was a very concrete example.

### Design Rationale
[Your synthesis - why this design, what tradeoffs, why alternatives fail]
The main reason that net protocols use big endian is simply because of how it was standardized by old machines, and that translated into RFCs. The tradeoff for this is that a host must translate their information into a network long before transmission, and translate it into a host long after receiving a message back. The only alternative would be for every network protocol to switch to using little endian like most modern CPUs. This would not be worth the risk and effort and it might be impossible to make sure little endian is used because no information in a byte says which endian it was designed for. Additionally, the benefits of little endian, faster math operations, do not apply as much for network protocols. The left-to-right intuitiveness of big-endian also would be sacrificed.

### Implementation Insight
[Your "aha moment" - how this changed your understanding]
I had a couple of important moments that changed my understanding. The first was simply reviewing what little vs big endian meant for processing bytes, the tradeoffs, and why big endian was first used. Seeing the different outputs between the functional program, and the program that did not convert between formats, was also important. It showed how crucial it is to switch between the formats or else the data is bad.
---

## Investigation 2: Topic 7: Fractional Seconds Representation

### Implementation Context
[What you coded and what puzzled you]
What I implemented was this function:
```
void get_current_ntp_time(ntp_timestamp_t *ntp_ts){
    struct timeval tv;
    gettimeofday(&tv, NULL);

    uint32_t ntp_seconds = tv.tv_sec + NTP_EPOCH_OFFSET;
    uint32_t ntp_fraction = (uint32_t)(((uint64_t)tv.tv_usec * NTP_FRACTION_SCALE) / USEC_INCREMENTS);

    ntp_ts->seconds = ntp_seconds;
    ntp_ts->fraction = ntp_fraction;
    printf("ntp_fraction: %u\n", ntp_fraction);
}
```

that converted microseconds to an NTP fraction scale.

What puzzled me was the following:
- Why use this scale instead of milliseconds or microseconds?
- Does using this scale provide more precision?
- What happens if you don't use the NTP scale?

### Investigation Journey  

#### Step 1. Gather information from slides about NTP scale

#### Step 2. Gather information from online about NTP scale vs microsecond scale

#### Step 3. Research the difference in precision and why it is used over microseconds?

#### Step 3. Run the program normally

#### Step 4. Run the program without converting (commenting out the conversion line)

#### Step 5. Compare results

#### Step 6. Research what happens if the NTP scale isn't used in other scenarios other than my program.

### Design Rationale
[Your synthesis - why this design, what tradeoffs, why alternatives fail]

### Implementation Insight
[Your "aha moment" - how this changed your understanding]