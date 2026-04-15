# DPI Engine - Deep Packet Inspection System v2.1

**Real-Time Network Packet Capture & Filtering with Deep Packet Inspection**

A high-performance, multi-threaded Deep Packet Inspection (DPI) engine that can process network traffic in two modes:

- **Offline Mode**: Analyze saved PCAP files (from Wireshark captures)
- **Live Mode**: Real-time packet capture from network interfaces with application classification and filtering

---

## Table of Contents

1. [What is DPI?](#what-is-dpi)
2. [Key Features](#key-features)
3. [System Requirements](#system-requirements)
4. [Installation & Compilation](#installation--compilation)
5. [Usage Guide](#usage-guide)
6. [Examples](#examples)
7. [Understanding the Architecture](#understanding-the-architecture)
8. [Output & Reports](#output--reports)
9. [Important Limitations](#important-limitations)
10. [Troubleshooting](#troubleshooting)
11. [Project Structure](#project-structure)

---

## What is DPI?

**Deep Packet Inspection (DPI)** is a technology that examines the contents of network packets beyond just their headers. While traditional firewalls only look at:

- Source/Destination IP addresses
- Ports
- Protocol (TCP/UDP)

Our DPI engine also inspects:

- **SNI (Server Name Indication)**: The domain name from HTTPS handshakes
- **Host headers**: From HTTP requests
- **Application signatures**: YouTube, Facebook, Google, TikTok, etc.

### Real-World Applications

- **ISP Traffic Management**: Throttle or prioritize certain applications
- **Enterprise Security**: Block social media on office networks
- **Parental Controls**: Restrict access to specific websites
- **Network Monitoring**: Understand what applications consume bandwidth
- **Compliance**: Monitor for policy violations

---

## Key Features

✅ **Multi-threaded Architecture**

- Load Balancer threads distribute traffic
- Fast Path threads perform parallel classification
- No single bottleneck - scales with CPU cores

✅ **Real-Time Packet Capture**

- Live capture from network interfaces (eth0, enp0s3, etc.)
- Graceful shutdown with Ctrl+C
- Stateful flow tracking across packets

✅ **Deep Application Classification**

- Extracts domain names from HTTPS Client Hello (SNI)
- Parses HTTP Host headers
- DNS traffic detection
- Port-based fallback classification

✅ **Flexible Blocking Rules**

- Block by IP address
- Block by application type
- Block by domain (substring matching)
- Rules loaded before processing

✅ **Comprehensive Statistics**

- Per-application traffic breakdown
- Thread utilization reporting
- Detected domain inventory
- Forwarded vs. dropped packet counts

---

## System Requirements

### Minimum

- **OS**: Linux (tested on Ubuntu 20.04+) or macOS
- **CPU**: Dual-core processor
- **RAM**: 2GB minimum (8GB+ recommended for live capture)
- **C++ Compiler**: GCC 7+ or Clang 5+ (with C++17 support)

### Dependencies

- **libpcap-dev**: For packet capture

  ```bash
  # Ubuntu/Debian
  sudo apt-get install libpcap-dev

  # macOS
  brew install libpcap
  ```

- **pthread**: Usually included with OS

### Network Permissions

- **Live capture requires root**: Use `sudo` to capture from network interfaces
- **File-based mode**: No special permissions needed

---

## Installation & Compilation

### Step 1: Clone/Download the Project

```bash
cd packet_analyzer
```

### Step 2: Verify Dependencies

```bash
# Check if libpcap is installed
pkg-config --cflags --libs libpcap

# Should output something like:
# -I/usr/include -L/usr/lib -lpcap
```

If not installed:

```bash
# Ubuntu/Debian
sudo apt-get install libpcap-dev

# macOS
brew install libpcap
```

### Step 3: Compile

**Standard compilation (with pthread support for multi-threading):**

```bash
g++ -std=c++17 -pthread -O2 -I include \
    src/dpi_mt.cpp src/pcap_reader.cpp src/packet_parser.cpp \
    src/sni_extractor.cpp src/types.cpp -lpcap -o dpi_engine
```

**Or using clang:**

```bash
clang++ -std=c++17 -pthread -O2 -I include \
    src/dpi_mt.cpp src/pcap_reader.cpp src/packet_parser.cpp \
    src/sni_extractor.cpp src/types.cpp -lpcap -o dpi_engine
```

**Verification:**

```bash
# Check if binary was created
ls -lh dpi_engine

# Should show something like:
# -rwxr-xr-x 1 user user 150K Jan 15 10:30 dpi_engine
```

---

## Usage Guide

### General Syntax

```
./dpi_engine <source> <output.pcap> [options]
```

### Parameters

| Parameter       | Description                                               |
| --------------- | --------------------------------------------------------- |
| `<source>`      | PCAP file (offline) OR interface name when using `--live` |
| `<output.pcap>` | Output file to save filtered packets                      |

### Available Options

```
--block-ip <ip>          Block traffic from this source IP
                         Example: --block-ip 192.168.1.50

--block-app <app>        Block specific application type
                         Valid apps: YouTube, Facebook, Google, TikTok, etc.
                         Example: --block-app YouTube

--block-domain <domain>  Block domain (substring match, case-insensitive)
                         Example: --block-domain youtube.com

--lbs <n>                Number of Load Balancer threads (default: 2)
                         Example: --lbs 4

--fps <n>                Fast Path threads per Load Balancer (default: 2)
                         Example: --fps 4

--live <interface>       Enable real-time capture mode (NOT file mode)
                         Specify network interface name
                         Example: --live eth0
```

### Network Interface Names

Common interface names by OS/platform:

| OS/Platform          | Typical Names                  |
| -------------------- | ------------------------------ |
| Linux (VM/Docker)    | eth0, enp0s3, ens0             |
| Linux (Raspberry Pi) | eth0, wlan0                    |
| Ubuntu/Debian        | enp0s* (Ethernet), wlp* (WiFi) |
| macOS                | en0, en1                       |
| Generic              | any (capture all interfaces)   |

**Find your interface:**

```bash
# Linux
ip link show

# macOS
ifconfig

# macOS alternative
networksetup -listallhardwareports
```

---

## Examples

### Example 1: Offline Mode - Basic Filtering

Process a saved PCAP file and save output (no rules):

```bash
./dpi_engine test_dpi.pcap filtered_output.pcap
```

**Output:**

```
╔══════════════════════════════════════════════════════════════╗
║          DPI ENGINE v2.1 (Multi-threaded + Live Capture)     ║
╠══════════════════════════════════════════════════════════════╣
║ Load Balancers:  2    FPs per LB:  2    Total FPs:  4        ║
╚══════════════════════════════════════════════════════════════╝

[Reader] Processing packets...
[Reader] Done reading 1245 packets
...
```

### Example 2: Offline Mode - Block YouTube

Block YouTube traffic from a PCAP file:

```bash
./dpi_engine input.pcap filtered.pcap \
    --block-domain youtube.com \
    --block-domain googlevideo.com \
    --block-domain ytimg.com
```

### Example 3: Offline Mode - Block Multiple Applications

Block YouTube, TikTok, and specific IP:

```bash
./dpi_engine input.pcap filtered.pcap \
    --block-app YouTube \
    --block-app TikTok \
    --block-ip 192.168.1.50
```

### Example 4: Live Capture Mode - Real-Time YouTube Blocking ⭐

Capture live traffic from eth0 and filter YouTube:

```bash
sudo ./dpi_engine eth0 live_output.pcap --live eth0 \
    --block-domain youtube.com \
    --block-domain googlevideo.com \
    --block-domain ytimg.com
```

**What happens:**

1. Capture starts immediately
2. Each packet is classified in real-time
3. YouTube packets are dropped (not written to output)
4. Other traffic is saved to `live_output.pcap`
5. Press Ctrl+C to stop gracefully

### Example 5: Live Capture Mode - Multi-Domain Blocking

Block multiple services in real-time:

```bash
sudo ./dpi_engine eth0 filtered_traffic.pcap --live eth0 \
    --block-domain facebook.com \
    --block-domain tiktok.com \
    --block-domain netflix.com \
    --block-ip 8.8.8.8
```

### Example 6: Custom Thread Configuration

Tune thread count for your CPU (more threads = better parallelism):

```bash
# For 8-core CPU: 4 LBs × 4 FPs per LB = 16 total processing threads
./dpi_engine input.pcap output.pcap --lbs 4 --fps 4
```

### Example 7: Live Capture to Specific Interface

Monitor a WiFi interface on Ubuntu:

```bash
sudo ./dpi_engine wlp2s0 capture_wifi.pcap --live wlp2s0
```

---

## Understanding the Architecture

### High-Level Flow

```
Real-Time Packets from eth0 (or file read)
          ↓
    ┌─────────────┐
    │   Reader    │ (Main thread reads packets)
    └──────┬──────┘
           │ (Hash-based distribution)
      ┌────┴────┐
      ↓         ↓
  ┌───────┐ ┌───────┐
  │  LB0  │ │  LB1  │ (Load Balancers distribute to FPs)
  └───┬───┘ └───┬───┘
   ┌──┴──┐  ┌──┴──┐
   ↓     ↓  ↓     ↓
 ┌──┐  ┌──┐┌──┐  ┌──┐
 │FP│  │FP││FP│  │FP│ (Fast Paths: classify, check rules, filter)
 └──┘  └──┘└──┘  └──┘
   └────┬─────────┘
        ↓
  ┌──────────────┐
  │Output Writer │ (Writes forwarded packets to output.pcap)
  └──────────────┘
```

### Why This Design?

1. **Consistent Hashing**: Each connection (5-tuple) always goes to the same FP
   - Ensures proper flow state tracking
   - Allows accurate SNI extraction across packets
2. **Load Balancer Layer**: Distributes work evenly across FPs
   - Prevents any single FP from becoming bottleneck
   - Scales with number of cores

3. **Output Queue**: Decouples packet processing from file I/O
   - Prevents slow disk writes from blocking FPs
   - Improves throughput

### Packet Journey

```
1. Packet Arrives
   │
   └─ Extract 5-tuple: (src_ip, dst_ip, src_port, dst_port, protocol)

2. Hash 5-tuple → Select Load Balancer
   │
   └─ LB hashes again → Select Fast Path

3. Fast Path Processes
   ├─ Look up flow in local hash table
   ├─ If HTTPS (port 443): Extract SNI from TLS Client Hello
   ├─ If HTTP (port 80): Extract Host header
   ├─ Classify application: YouTube? Facebook? DNS?
   └─ Check blocking rules

4. Decision
   ├─ If BLOCKED: Increment dropped counter, don't forward
   └─ If ALLOWED: Push to output queue

5. Output Writer
   ├─ Wait for forwarded packets
   ├─ Write PCAP header + data to file
   └─ Repeat until shutdown
```

---

## Output & Reports

### Console Output Example

```
╔══════════════════════════════════════════════════════════════╗
║          DPI ENGINE v2.1 (Multi-threaded + Live Capture)     ║
╠══════════════════════════════════════════════════════════════╣
║ Load Balancers:  2    FPs per LB:  2    Total FPs:  4        ║
╚══════════════════════════════════════════════════════════════╝

[Rules] Blocked domain: youtube.com
[Rules] Blocked domain: googlevideo.com
[Rules] Blocked domain: ytimg.com

[Live Capture] Successfully opened interface: eth0
[Live Capture] Starting real-time capture on eth0 (Ctrl+C to stop)...

[SNI Detected] www.youtube.com | App: YouTube | SrcPort: 54321
[BLOCKED] www.youtube.com (App: YouTube)
[SNI Detected] www.google.com | App: Google | SrcPort: 54322

^C
[INFO] Shutdown signal received (Ctrl+C). Draining queues and stopping...
[Live Capture] Stopped. Processed 2847 packets

╔══════════════════════════════════════════════════════════════╗
║                      PROCESSING REPORT                        ║
╠══════════════════════════════════════════════════════════════╣
║ Total Packets:               2847                             ║
║ Total Bytes:              892345                              ║
║ TCP Packets:               2780                               ║
║ UDP Packets:                 67                               ║
╠══════════════════════════════════════════════════════════════╣
║ Forwarded:                  2620                              ║
║ Dropped:                     227                              ║
╠══════════════════════════════════════════════════════════════╣
║ THREAD STATISTICS                                             ║
║   LB0 dispatched:          1450                               ║
║   LB1 dispatched:          1397                               ║
║   FP0 processed:            720                               ║
║   FP1 processed:            680                               ║
║   FP2 processed:            730                               ║
║   FP3 processed:            717                               ║
╠══════════════════════════════════════════════════════════════╣
║                   APPLICATION BREAKDOWN                       ║
╠══════════════════════════════════════════════════════════════╣
║ HTTPS                 1200  42.2% ################################
║ YouTube                 227   8.0% ###### (BLOCKED)
║ Google                  150   5.3% ####
║ DNS                     120   4.2% ###
║ Facebook                 80   2.8% ##
║ Unknown                 970  34.1% ##########################
╚══════════════════════════════════════════════════════════════╝

[Detected Domains/SNIs]
  - www.youtube.com -> YouTube
  - www.google.com -> Google
  - www.facebook.com -> Facebook
  - github.com -> Unknown
  - accounts.google.com -> Google

Output written to: live_output.pcap
```

### Understanding the Report

| Section                   | What It Shows                                   |
| ------------------------- | ----------------------------------------------- |
| **Configuration**         | Number of threads and processing pipeline setup |
| **Rules**                 | Active blocking rules loaded                    |
| **Total Packets**         | All packets processed (forwarded + dropped)     |
| **Forwarded**             | Packets that passed the filter (allowed)        |
| **Dropped**               | Packets blocked by rules                        |
| **Thread Statistics**     | Load distribution across threads                |
| **Application Breakdown** | Traffic classification with percentages         |
| **Detected Domains**      | SNI/Host values found with their classification |

---

## Important Limitations

### ⚠️ Important: This is NOT Real-Time Blocking

This engine captures packets and **writes them to a file** based on filtering rules. It does **NOT block packets in real-time** at the kernel level. Here's why:

```
What This Does:
┌──────────────┐
│ Packet from  │
│ eth0 arrives │
└──────┬───────┘
       │
       ▼
┌───────────────────────┐
│ Classify application  │
└──────┬────────────────┘
       │
       ▼
┌──────────────────────────────┐
│ Check if blocked:            │
│ ✓ Allowed → Write to file    │
│ ✗ Blocked → Don't write      │
└──────────────────────────────┘
       │
       ▼
   Output PCAP
  (for analysis)

Problem: User-space application can't actually block
         the original packet from reaching the destination!
```

### Why Kernel-Level Implementation is Needed for Real Blocking

Real-time blocking requires:

1. **eBPF/XDP programs**: Kernel-level packet filtering
2. **Netfilter/iptables hooks**: At the kernel network stack
3. **Network drivers**: Interception before packets reach applications

Examples:

- **Linux**: eBPF/XDP, netfilter, iptables
- **Windows**: WinDivert, Windows Filtering Platform
- **macOS**: pfctl, packet filter

### What This Engine IS Good For

✅ **Network Monitoring & Analysis**

- See what applications are running
- Understand traffic distribution
- Create filtered PCAP files for Wireshark analysis

✅ **Security Research**

- Study DPI techniques
- Learn SNI extraction
- Understand packet classification

✅ **Network Auditing**

- Generate reports of application usage
- Detect unauthorized applications
- Compliance monitoring

✅ **Data Reduction**

- Filter out unimportant traffic
- Keep only relevant packets for analysis
- Reduce storage requirements

### Real Blocking Use Cases (Requires Kernel)

❌ **Parental Controls** - Need kernel-level filtering
❌ **ISP Throttling** - Need QoS at kernel level
❌ **DPI Firewall** - Need packet interception at kernel
❌ **Network Gateway** - Need packet rewriting at kernel

---

## Troubleshooting

### Issue: "Cannot open output file"

**Cause**: No write permission in target directory

**Solution**:

```bash
# Use a writable directory
./dpi_engine input.pcap /tmp/output.pcap

# Or use relative path
./dpi_engine input.pcap ./output.pcap

# Check permissions
ls -ld .
```

### Issue: Compilation Error - "libpcap not found"

**Cause**: libpcap development files not installed

**Solution**:

```bash
# Ubuntu/Debian
sudo apt-get install libpcap-dev

# Verify installation
pkg-config --cflags --libs libpcap

# Try compiling again
g++ -std=c++17 -pthread -O2 -I include \
    src/dpi_mt.cpp src/pcap_reader.cpp src/packet_parser.cpp \
    src/sni_extractor.cpp src/types.cpp -lpcap -o dpi_engine
```

### Issue: Live Capture - "Permission denied" on eth0

**Cause**: Need root privileges to capture packets

**Solution**:

```bash
# Use sudo
sudo ./dpi_engine eth0 output.pcap --live eth0

# Or grant CAP_NET_RAW to binary (less secure)
sudo setcap cap_net_raw+p ./dpi_engine
./dpi_engine eth0 output.pcap --live eth0
```

### Issue: "Unknown interface: eth0"

**Cause**: Interface name incorrect or doesn't exist

**Solution**:

```bash
# List all available interfaces
ip link show          # Linux
ifconfig             # macOS/Linux
networksetup -listallhardwareports  # macOS

# Try 'any' to capture from all interfaces
sudo ./dpi_engine any output.pcap --live any
```

### Issue: No packets captured in live mode

**Cause 1**: Interface has no traffic

```bash
# Generate test traffic
ping 8.8.8.8
# In another terminal:
sudo ./dpi_engine eth0 output.pcap --live eth0
```

**Cause 2**: Interface not in promiscuous mode

```bash
# Check if packets are received
ifconfig eth0
# Or use tcpdump to verify
sudo tcpdump -i eth0 -c 5
```

### Issue: High CPU usage

**Cause**: Too many threads competing for resources

**Solution**:

```bash
# Reduce thread count (default is 2 LBs × 2 FPs = 4 threads)
./dpi_engine input.pcap output.pcap --lbs 1 --fps 1

# Or increase if you have more CPU cores
./dpi_engine input.pcap output.pcap --lbs 4 --fps 4
```

### Issue: SNI not being extracted

**Cause**: Packets don't contain TLS Client Hello

**Solution**:

```bash
# Verify packets are HTTPS (port 443)
# SNI is only extracted from first packet of connection
# If you capture mid-connection, SNI might not appear

# Use complete PCAP with full TLS handshake
# Or check for HTTP Host headers on port 80
```

---

## Project Structure

```
packet_analyzer/
├── include/                           # Header files (declarations)
│   ├── pcap_reader.h                 # PCAP file/live capture reading
│   ├── packet_parser.h               # Protocol header parsing
│   ├── sni_extractor.h               # TLS SNI & HTTP Host extraction
│   ├── types.h                       # Data structures (FiveTuple, AppType, etc.)
│   ├── thread_safe_queue.h           # Thread-safe queue for IPC
│   └── dpi_engine.h                  # Main orchestrator
│
├── src/                              # Implementation files
│   ├── pcap_reader.cpp               # PCAP reading (file & live)
│   ├── packet_parser.cpp             # Network protocol parsing
│   ├── sni_extractor.cpp             # SNI/Host extraction logic
│   ├── types.cpp                     # Helper functions & data types
│   ├── dpi_mt.cpp                    # ★ Multi-threaded main program
│   ├── main_working.cpp              # Simple single-threaded version (learning)
│   └── [other files]                 # Supporting implementation
│
├── README.md                          # This file
├── generate_test_pcap.py             # Creates test capture data
├── test_dpi.pcap                     # Sample capture file
└── dpi_engine                        # Compiled binary (after build)
```

### Key Components Explained

**dpi_mt.cpp** (Multi-threaded Main)

- Reader thread: Reads from PCAP file or live interface
- Load Balancer threads: Distribute packets to Fast Path threads
- Fast Path threads: Perform classification and filtering
- Output Writer thread: Writes filtered packets to output PCAP
- Uses hash-based consistent routing for flow state tracking

**packet_parser.cpp**

- Extracts Ethernet headers (MAC addresses)
- Parses IPv4 headers (IPs, protocol)
- Parses TCP/UDP headers (ports, flags)
- Handles variable-length headers correctly

**sni_extractor.cpp**

- Detects TLS Client Hello packets (HTTPS handshake)
- Navigates through TLS record structure
- Finds SNI extension in TLS extensions
- Extracts domain name string
- Also extracts HTTP Host headers from plain HTTP

**types.cpp**

- FiveTuple: Represents a connection (src_ip, dst_ip, src_port, dst_port, protocol)
- AppType: Enumeration of known applications
- sniToAppType(): Maps domains to application types
- FiveTupleHash: Hash function for consistent hashing

---

## Advanced Topics

### Modifying Application Signatures

To add blocking for a new application, edit `src/types.cpp`:

```cpp
AppType sniToAppType(const std::string& sni) {
    std::string lower = sni;
    std::transform(lower.begin(), lower.end(), lower.begin(), ::tolower);

    // Add your new application
    if (lower.find("twitch.tv") != std::string::npos)
        return AppType::TWITCH;

    if (lower.find("tiktok.com") != std::string::npos)
        return AppType::TIKTOK;

    // ... existing code
}
```

### Performance Tuning

For optimal performance on your system:

```bash
# Count CPU cores
nproc  # Linux
sysctl -n hw.ncpu  # macOS

# Set threads = number of cores for best throughput
./dpi_engine input.pcap output.pcap --lbs 4 --fps 4
# This creates 4 LBs × 4 FPs = 16 worker threads
```

### Understanding the Five-Tuple

Every connection is uniquely identified by 5 values:

```
Source IP:      192.168.1.100   (Your device)
Destination IP: 142.250.185.206 (YouTube server)
Source Port:    54321           (Ephemeral port)
Dest Port:      443             (HTTPS)
Protocol:       TCP (6)         (TCP or UDP)

Hash of this 5-tuple determines which FP processes all packets
of this connection → ensures SNI from any packet reaches same FP
```

---

## FAQs

**Q: Will this actually block YouTube on my network?**
A: Not directly in real-time. This engine **filters the PCAP output**, which is useful for analysis. For actual blocking, you need kernel-level tools (iptables, eBPF, pfctl, etc.).

**Q: How do I measure if blocking is working?**
A: Compare input vs output PCAP file sizes. Blocked traffic won't appear in output:

```bash
ls -lh input.pcap output.pcap
# input.pcap should be larger than output.pcap if blocking is active
```

**Q: Can I run this without root?**
A: Yes for file-based mode. Live capture requires `sudo` or CAP_NET_RAW capability.

**Q: Which applications can be detected?**
A: Any application using HTTPS (port 443) with SNI, or HTTP (port 80) with Host headers. Currently detects: YouTube, Facebook, Google, TikTok, GitHub, Netflix, and more.

**Q: Does this detect encrypted traffic (QUIC, HTTP/3)?**
A: QUIC (UDP port 443) is not yet supported. QUIC SNI is encrypted differently. This is a future enhancement.

**Q: How many packets per second can it handle?**
A: On a modern 8-core CPU, approximately 50,000-100,000 packets/second (varies by packet size and classification complexity).

---

## Getting Started

### Quick Start (5 minutes)

1. **Compile:**

   ```bash
   g++ -std=c++17 -pthread -O2 -I include \
       src/dpi_mt.cpp src/pcap_reader.cpp src/packet_parser.cpp \
       src/sni_extractor.cpp src/types.cpp -lpcap -o dpi_engine
   ```

2. **Test with sample file:**

   ```bash
   ./dpi_engine test_dpi.pcap output.pcap
   ```

3. **Try live capture:**

   ```bash
   sudo ./dpi_engine eth0 live_capture.pcap --live eth0
   ```

4. **View results:**
   ```bash
   # Open output.pcap in Wireshark
   wireshark output.pcap
   ```

---

## License & Attribution

This is a **demonstration project** for learning network programming, packet parsing, and multi-threaded systems design. Not intended for production use without proper security auditing.

---

## Contact & Support

For questions about:

- **Architecture**: See the detailed documentation at the start of `src/dpi_mt.cpp`
- **SNI Extraction**: Review `src/sni_extractor.cpp` and the TLS specification
- **Compilation Issues**: Check system libpcap installation
- **Live Capture**: Verify network interface names with `ip link show` or `ifconfig`

Happy learning! 🚀

---

**Last Updated**: January 2025  
**Version**: 2.1 (Real-Time Capture + Multi-threaded)
