# Deep Packet Inspection System

**Deep Packet Inspection (DPI) Engine** reads packets from a PCAP capture, parses supported network protocols, identifies application traffic from packet metadata, applies filtering rules, and writes allowed packets to a new PCAP file. It includes a straightforward single-threaded implementation and a multi-threaded implementation.


## Prerequisites

- A C++17 compiler:
  - Windows: Visual Studio 2022 Build Tools (MSVC) or MinGW-w64 with GCC 7+
  - Linux/macOS: GCC 7+ or Clang with C++17 library support
- Python 3, only if you want to generate the sample PCAP with `generate_test_pcap.py`
- A PCAP input file. The repository includes `test_dpi.pcap` for local testing.

No third-party C++ libraries are required for the manual build commands below.

## Project Structure

```text
.
├── include/                  # Public declarations and shared types
│   ├── connection_tracker.h  # Flow tracking
│   ├── dpi_engine.h          # DPI engine interface
│   ├── fast_path.h           # Multi-threaded packet processing
│   ├── load_balancer.h       # Work distribution
│   ├── packet_parser.h       # Protocol parsing
│   ├── pcap_reader.h         # PCAP input
│   ├── platform.h            # Platform-specific definitions
│   ├── rule_manager.h        # Blocking rules
│   ├── sni_extractor.h       # Application metadata extraction
│   ├── thread_safe_queue.h   # Inter-thread queues
│   └── types.h               # Shared packet and flow types
├── src/                      # Implementations and executable entry points
│   ├── connection_tracker.cpp
│   ├── dpi_engine.cpp
│   ├── dpi_mt.cpp            # Multi-threaded executable
│   ├── fast_path.cpp
│   ├── load_balancer.cpp
│   ├── main.cpp
│   ├── main_dpi.cpp
│   ├── main_simple.cpp
│   ├── main_working.cpp      # Single-threaded DPI executable
│   ├── packet_parser.cpp
│   ├── pcap_reader.cpp
│   ├── rule_manager.cpp
│   ├── sni_extractor.cpp
│   └── types.cpp
├── generate_test_pcap.py    # Generates sample packet data
├── test_dpi.pcap             # Sample input capture
├── CMakeLists.txt
└── WINDOWS_SETUP.md          # Additional Windows setup guidance
```

## Architecture

The engine processes saved PCAP files; it does not capture live network traffic.

```text
Input PCAP
    │
    ▼
Read packet → Parse Ethernet/IP/TCP/UDP → Track flow by five-tuple
                                              │
                                Inspect available application metadata
                                              │
                                Apply IP/app/domain blocking rules
                                      ┌───────┴────────┐
                                      ▼                ▼
                                  Drop packet     Write packet
                                                       │
                                                       ▼
                                                  Output PCAP
```

The single-threaded implementation performs these steps sequentially. The multi-threaded implementation distributes packets through a reader, load balancers, fast-path workers, and an output writer. Hashing by flow keeps packets from a connection on the same worker, while thread-safe queues pass work between stages.

## Tools, Technologies, and Concepts

- **C++17:** Main implementation language; uses the standard library for threading and synchronization.
- **PCAP:** File format used for packet input and output.
- **Python 3:** Optional test-capture generator.
- **CMake:** Project build configuration (`CMakeLists.txt`).
- **Networking:** Ethernet, IPv4, TCP/UDP, ports, and the five-tuple used to identify a flow.
- **Inspection and classification:** TLS ClientHello SNI and available HTTP host metadata are used to identify domains/applications. The engine inspects metadata; it does not decrypt TLS application data.
- **Filtering:** Rules can match a source IP, application, or domain.
- **Concurrency:** Reader/worker/writer pipeline, consistent flow hashing, producer-consumer queues, mutexes, and condition variables.

## Implementation

The main processing components are:

- `PcapReader` reads packet records from a capture.
- `PacketParser` extracts supported link/network/transport fields.
- `SNIExtractor` inspects application metadata for hostnames.
- Flow tracking associates packets with connection state.
- The rule manager checks configured IP, application, and domain rules.
- The single-threaded entry point is `src/main_working.cpp`; the multi-threaded entry point is `src/dpi_mt.cpp`.

The multi-threaded executable accepts `--lbs` for the number of load balancers and `--fps` for fast-path workers per load balancer. Both DPI executables support `--block-ip`, `--block-app`, and `--block-domain`.

## Build and Run

Run commands from the repository root.

### Build with GCC or Clang

Single-threaded:

```bash
g++ -std=c++17 -O2 -I include -o dpi_simple \
  src/main_working.cpp src/pcap_reader.cpp src/packet_parser.cpp \
  src/sni_extractor.cpp src/types.cpp
```

Multi-threaded:

```bash
g++ -std=c++17 -pthread -O2 -I include -o dpi_engine \
  src/dpi_mt.cpp src/pcap_reader.cpp src/packet_parser.cpp \
  src/sni_extractor.cpp src/types.cpp
```

On Windows with MinGW-w64, use the same source lists and compiler options, changing the output names to `dpi_simple.exe` and `dpi_engine.exe` if desired. With MSVC, run from a Visual Studio Developer Command Prompt. For the single-threaded executable:

```cmd
cl /EHsc /std:c++17 /O2 /I include /Fe:dpi_simple.exe ^
  src\main_working.cpp src\pcap_reader.cpp src\packet_parser.cpp ^
  src\sni_extractor.cpp src\types.cpp
```

For the multi-threaded executable:

```cmd
cl /EHsc /std:c++17 /O2 /I include /Fe:dpi_engine.exe ^
  src\dpi_mt.cpp src\pcap_reader.cpp src\packet_parser.cpp ^
  src\sni_extractor.cpp src\types.cpp
```

The repository's `CMakeLists.txt` currently defines the `packet_analyzer` target from `main.cpp`, `pcap_reader.cpp`, and `packet_parser.cpp`. Use the explicit commands above to build either DPI executable.

### Generate sample input (optional)

```bash
python3 generate_test_pcap.py
```

On Windows, use `python generate_test_pcap.py` if that is the Python 3 command on your system.

### Run

Basic processing:

```bash
./dpi_engine test_dpi.pcap output.pcap
```

With filtering rules and worker configuration:

```bash
./dpi_engine test_dpi.pcap output.pcap \
  --block-app YouTube \
  --block-ip 192.168.1.50 \
  --block-domain facebook \
  --lbs 2 \
  --fps 2
```

On Windows, run `dpi_engine.exe` instead of `./dpi_engine`. The program writes packets that pass the rules to the output PCAP and reports processing statistics to the console.
