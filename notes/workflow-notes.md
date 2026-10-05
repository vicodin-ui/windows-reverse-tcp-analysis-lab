## Lab Workflow Notes

These notes are for the geeks and document the workflow and concepts studied during the Windows Reverse TCP Analysis Lab.

All testing was preformed between user-owned virtual machines on an isolated VirtualBix Internal Network.

## Lab Systems

| System | Role | IP Adress |
| --- | --- | --- |
| Kali Linux | Analysis / server system | `10.10.10.10` |
| Windows-Lab | Windows client | `10.10.10.20` |

Internal VirtualBox Network:

`cyberlab`

No default gateway or DNS server was required for the isolated lab.

---

## 1. Reverse TCP Concept

A reverse TCP connection works differently from a normal inbound connection.

Instead of the controlling/listening system initiating the connection, the client initiates an outbound TCP connection toward a configured listener in this case Metasploits default handler.

Conceptually:

Windows client
    |
    | outbound TCP connection
    v
Kali listener

Important terms:

- `LHOST` = IP address the client is configured to connect back to.
- `LPORT` = TCP port used for that connection.
- The client initiates the TCP connection.
- The handler waits for and accepts that connection.

This means payload generation and connection are separate stages.

---

## 2. Generated Lab File

Metasploit tooling was used to generate a Windows reverse TCP lab artifact.

The generated configuartion contained:

- Target platform: Windows
- Connection type: Reverse TCP
- Callback host: Kali system
- Callback port: `4444`

The key takeaway is that the callback destination is configured before execution.

Generating the file does NOT:

- start the listener
- transfer the file
- execute the file on Windows
- automatically create a connection

Those are separate stages.

NO generated payload file are stored in this repository.

---

## 3. HTTP Delivery

A temp Python HTTP server was used to make files avaliable to the Windows VM.

Command:

```bash
python3 -m http.server 80

## 4. Metasploit Handler

`msfconsole`

opens the Metasploit Framework command-line interface.

It does NOT automatically start a listener.

The generic handler is then selected and configured for the same
connection type expected from the Windows client.

Conceptually:

```text
msfconsole
    |
    v
multi/handler
    |
    v
matching reverse TCP session type
    |
    v
configure LHOST
    |
    v
configure LPORT
    |
    v
start handler
```

Once started, the handler listens on the configured local IP address
and TCP port.

Example:

```text
10.10.10.10:4444
```

The handler waits for the Windows client to initiate the reverse TCP
connection.

The handler does NOT execute anything on Windows.

---

## 5. Windows Defender Detection

During testing, Windows Defender detected the generated lab artifact.

Instead of treating the detection as something to bypass, the lab
shifted toward understanding what Defender observed.

Topics studied:

- Endpoint detection
- File scanning
- Malware classification
- Detection events
- Quarantine and remediation
- Difference between file creation, execution, and network activity

This became the transition from reverse TCP workflow study into
defensive analysis.

---

## 6. Wireshark Capture

Wireshark was used to observe traffic on the Kali system's `eth0`
interface.

Windows:

`10.10.10.20`

Kali:

`10.10.10.10`

A Windows ping generated ICMP Echo Requests.

Wireshark captured the corresponding replies from Kali.

Example:

```text
10.10.10.20 -> 10.10.10.10    ICMP Echo Request
10.10.10.10 -> 10.10.10.20    ICMP Echo Reply
```

Windows sends four Echo Requests by default when using `ping`.

---

## 7. Wireshark Interface

Wireshark separates packet analysis into three main areas.

### Packet List

The top pane shows captured packets.

Useful columns include:

- Time
- Source
- Destination
- Protocol
- Length
- Info

### Packet Details

The bottom-left pane decodes the selected packet into human-readable
protocol fields.

Example:

```text
Ethernet II
    IPv4
        ICMP
```

### Raw Packet Bytes

The bottom-right pane contains the actual captured packet bytes.

Therefore:

```text
Left  = decoded packet structure
Right = raw packet bytes
```

Selecting a decoded field highlights the matching raw bytes.

---

## 8. IPv4 Header Analysis

An ICMP Echo Request from Windows contained fields including:

- Version
- Header Length
- Total Length
- Identification
- Flags
- Fragment Offset
- TTL
- Protocol
- Header Checksum
- Source Address
- Destination Address

Example:

```text
Source Address:      10.10.10.20
Destination Address: 10.10.10.10
TTL:                 128
Protocol:            ICMP (1)
```

The addresses could also be observed directly in hexadecimal:

```text
0a 0a 0a 14 = 10.10.10.20
0a 0a 0a 0a = 10.10.10.10
```

Where:

```text
0a = 10
14 = 20
```

This demonstrated that Wireshark reads the IP addresses directly from
the IPv4 header.

---

## 9. Encapsulation

The captured ping demonstrated encapsulation.

Conceptually:

```text
Ethernet
    |
    +-- IPv4
            |
            +-- ICMP
                    |
                    +-- Data
```

One captured frame contained:

```text
Ethernet header = 14 bytes
IPv4 packet     = 60 bytes
-------------------------
Ethernet frame  = 74 bytes
```

The IPv4 packet contained:

```text
IPv4 header     = 20 bytes
ICMP + data     = 40 bytes
-------------------------
IPv4 packet     = 60 bytes
```

This made encapsulation visible in actual captured traffic.

---

## 10. TTL

The Windows ICMP requests showed:

```text
TTL = 128
```

The Kali responses showed:

```text
TTL = 64
```

These are common starting TTL values for Windows and Linux systems.

Routers decrement the TTL field when forwarding IPv4 packets.

Because both systems were on the same VirtualBox Internal Network,
there was no router between them to decrement the TTL.

---

## 11. IPv4 Protocol Field

Inside the IPv4 header:

```text
Protocol: ICMP (1)
```

The Protocol field tells the receiving system what protocol is carried
inside the IPv4 packet.

Common examples:

```text
1  = ICMP
6  = TCP
17 = UDP
```

Conceptually:

```text
Ethernet
    -> IPv4
        -> Protocol 1
            -> ICMP
                -> Echo Request
```

---

## 12. ARP

Wireshark also captured ARP traffic.

Example:

```text
Who has 10.10.10.10?
Tell 10.10.10.20
```

The system owning `10.10.10.10` responds with its MAC address.

ARP provides the mapping:

```text
IPv4 Address <-> MAC Address
```

Local Ethernet delivery uses MAC addresses, so ARP allows a system to
discover which MAC address belongs to a local IPv4 address.

---

## Key Lessons

The workflow can be separated into:

```text
Generation
    ->
Delivery
    ->
Execution
    ->
Outbound connection
    ->
Listener / handler
    ->
Endpoint detection
    ->
Network analysis
```

Major lessons:

- Generating a file does not execute it
- Hosting a file does not push it to another machine
- An HTTP server and a TCP listener perform different jobs
- `msfconsole` is the Metasploit interface, not the listener itself
- Reverse TCP means the client initiates the connection
- Endpoint detection and network analysis provide different views of activity
- Wireshark translates raw packet bytes into readable protocol fields
- Encapsulation can be observed directly in packet captures
