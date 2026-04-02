# O-RAN E2T Fuzzer

@inproceedings{eshaghshafaei2026oran,
  title={High-Speed Fuzzing of O-RAN E2 Termination via
Component Isolation},
  author={Eshagh Shafaei},
  booktitle={IEEE [Conference Name]},
  year={2026}
}

[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![Fuzzer](https://img.shields.io/badge/Fuzzer-AFL++-orange.svg)](https://aflplus.plus/)

This repository contains the isolated fuzzing harness and methodology used to stress-test the O-RAN Near-RT RIC E2 Termination (E2T) service. 

By utilizing an **AFL++ persistent-mode harness with shared-memory I/O** and stripping artificial `assert()` barriers from the ASN.1 library, this setup achieves up to **23,000 executions per second** without the overhead of a full Kubernetes deployment.

This methodology resulted in the discovery of 6 zero-day vulnerabilities in the E2T service, including critical Heap Use-After-Free and Stack Buffer Overflow conditions.

## 🚨 Discovered Vulnerabilities

| Bug ID | Vulnerability Type (CWE) | Affected Component | Impact |
|--------|--------------------------|--------------------|--------|
| **1** | Null-Pointer Deref. (CWE-476) | `sctpThread.cpp` (Prometheus) | DoS (Crash) |
| **3** | Stack Use-After-Scope (CWE-908) | E2T Event Loop | Memory Corruption |
| **4** | Stack Buffer Overflow (CWE-121) | E2T Event Loop | Stack Corruption |
| **5** | Heap Use-After-Free (CWE-416) | `ConnectedCU_t` State Machine | RCE / Corruption |
| **6** | Heap Buffer Overflow (CWE-122)| `RmrMessagesBuffer_t` | Heap Corruption |
*(Bug #2 was a confirmed finding previously identified by ORANalyst).*

## 🛠️ Architecture
Unlike prior work that relies on network-bound Kubernetes deployments (<10 exec/s), this harness isolates the ASN.1 decoder and SCTP event loops in memory. Payloads are fed directly into the decoder via RAM, bypassing `fork()` overhead entirely.

## 🚀 Quick Start (Docker)
We strongly recommend using the provided Docker container to ensure all LLVM and AFL++ dependencies are correctly matched.

```bash
# 1. Build the fuzzer image
docker build -t e2t-fuzzer ./docker

# 2. Run the container
docker run -it --privileged e2t-fuzzer /bin/bash

# 3. Start the fuzzing campaign
./scripts/run_fuzzer.sh
