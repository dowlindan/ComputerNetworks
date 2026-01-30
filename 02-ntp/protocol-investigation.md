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
and the helper functio ntp_ts_to_net, as well as the inverse function which behaved the same.


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
- Older machines standardized big endian, and most RFs were created when machines were big endian
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
#### Step 5. Run the program WITHOUT converting between endians
#### Step 6. Contrast the results 

### Design Rationale
[Your synthesis - why this design, what tradeoffs, why alternatives fail]

### Implementation Insight
[Your "aha moment" - how this changed your understanding]

---

## Investigation 2: Topic 7: Fractional Seconds Representation

### Implementation Context
[What you coded and what puzzled you]

### Investigation Journey
[Your research/exploration process]

### Design Rationale
[Your synthesis in your own words]

### Implementation Insight
[What you learned from implementing]