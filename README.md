# O-RAN E2T Stateful Fuzzer

[](https://opensource.org/licenses/Apache-2.0)
[](https://aflplus.plus/)

Payloads and direct code for running the payload is going to add soon

This repository contains the isolated, **stateful fuzzing harness** and methodology used to stress-test the O-RAN Near-RT RIC E2 Termination (E2T) service.

By utilizing an **AFL++ persistent-mode harness with a 2-Phase state injection loop**, this setup bypasses the initial connection handshake, allowing the fuzzer to mutate payloads deep inside the active state machine (e.g., `RICserviceUpdate`, `RICcontrolRequest`). This methodology achieves **\~5,000 to 8,000 stateful executions per second** without the massive overhead of a full Kubernetes deployment.

## 📚 Reference

More information about this stateful fuzzing methodology is available in our research paper [Add Link Here]. The preprint is also available on [arXiv](Add Link Here). If you use this code or methodology in your scientific work, please cite the paper as follows:

```bibtex
@inproceedings{eshaghshafaei2026oran,
  title={Stateful High-Speed Fuzzing of O-RAN E2 Termination via Component Isolation},
  author={Eshagh Shafaei},
  booktitle={IEEE [Conference Name]},
  year={2026}
}
```

## 🚨 Discovered Vulnerabilities

This methodology filters out stateless false positives and has successfully identified critical zero-day vulnerabilities in the O-RAN E2T routing logic and ASN.1 parsers:

| ID | Vulnerability Type (CWE)           | Triggering Payload / Condition                                                   | Location / Status                         | Discovery Phase        |
|----|-----------------------------------|----------------------------------------------------------------------------------|------------------------------------------|------------------------|
| 1  | Null-Pointer Deref. (CWE-476)     | Out-of-sequence InitiatingMessage.                                               | asnInitiatingRequest() (Partially New)   | Phase 1: Stateless     |
| 2  | Null-Pointer Deref. (CWE-476)     | Out-of-sequence SuccessfulOutcome.                                               | asnSuccessfulMsg() (Partially New)       | Phase 1: Stateless     |
| 3  | Null-Pointer Deref. (CWE-476)     | Malformed UnsuccessfulOutcome (e.g., RICsubscriptionDeleteFailure).              | asnUnSuccsesfulMsg() (Novel Zero-Day)    | Phase 1: Stateless     |
| 4  | Unhandled Exception (CWE-248)     | E2SetupRequest containing illegal characters in gNB ID string.                  | prometheus-cpp registry (Known)          | Phase 1: Stateless     |
| 5  | Stack Use-After-Scope (CWE-908)   | Rapid, sequential delivery of crash primitives overwhelming recovery.           | sctpThread.cpp:841 (New)                 | Phase 2: Sequential    |
| 6  | Stack Buffer Overflow (CWE-121)   | Rapid, sequential delivery of invalid crash primitives.                         | sctpThread.cpp:852 (New)                 | Phase 2: Sequential    |
| 7  | Heap Buffer Overflow (CWE-122)    | Rapid, sequential delivery of invalid crash primitives.                         | sctpThread.cpp:877 (New)                 | Phase 2: Sequential    |
| 8  | Heap Use-After-Free (CWE-416)     | High-volume, concurrent connection flooding.                                    | sctpThread.cpp:1472 (New)                | Phase 2: Concurrent    |

## 🛠️ Architecture: The 2-Phase Stateful Harness

Unlike prior work that relies on slow, network-bound Kubernetes deployments (\<10 exec/s) or stateless harnesses that fail to reach deep protocol logic, this harness isolates the ASN.1 decoder and SCTP event loops in memory using a **2-Phase Loop**:

1.  **Phase 1 (State Injection):** The harness injects a valid, hardcoded `E2SetupRequest` directly into the E2T memory. This tricks the server into thinking a radio node has successfully connected, unlocking the deeper state machine.
2.  **Phase 2 (Weaponized Fuzzing):** The harness immediately feeds AFL++'s mutated payloads into the now-active connection, attacking deep handlers.
3.  **Phase 3 (State Reset):** The harness intercepts the telemetry loops, wipes the `ConnectedCU_t` structures, and resets the memory for the next loop to prevent artificial memory exhaustion.

*(Note: We deliberately strip the `-DUNIT_TEST` compiler flag to prevent artificial Null Pointer Dereferences that do not exist in the production binary).*

-----

## 🚀 Quick Start 

### Option A: Using Docker (Recommended)
We strongly recommend using the provided Docker container. The `Dockerfile` automatically downloads the required O-RAN RMR routing dependencies and compiles the C++ stateful fuzzer using the correct AddressSanitizer flags.

**1. Build the fuzzer image (This will auto-compile the C++ harness)**
```bash
docker build -t e2t-fuzzer -f docker/Dockerfile .

# 2. Run the container
docker run -it --privileged e2t-fuzzer /bin/bash

# 3. Start the fuzzing campaign
./scripts/run_fuzzer.sh
```

### Option B: Local Build

If you are building locally with AFL++ installed, simply use the provided Makefile. The Makefile dynamically targets your `afl-clang-fast++` compiler.

```bash
cd src/
make
```

*(If your AFL++ installation is not in your PATH, run `make CXX=/path/to/afl-clang-fast++`)*

-----

## 🔬 Running the Campaign & Triaging

### 1\. Running the Fuzzer

To prevent the fuzzer from falsely categorizing slow memory leaks (like the 184-byte ASN.1 leak) as fatal crashes, the fuzzer must be run with specific AddressSanitizer options and the `-m none` flag to remove artificial memory ceilings.

Use the provided script:

```bash
./scripts/run_fuzzer.sh
```

*(This executes: `ASAN_OPTIONS="detect_leaks=0:abort_on_error=1" afl-fuzz -m none -i results/initial_corpus -o results/out_stateful -- ./src/fuzz_sctpthread_target`)*

### 2\. Triaging Real Crashes

AFL++ may occasionally save loop-exhaustion artifacts. **Do not test fuzzer crashes against the live, uninstrumented E2T server.** To verify if a crash is a genuine memory corruption (e.g., Segfault, Buffer Overflow, UAF), pass it through the ASan-instrumented harness manually using the triage script:

```bash
./scripts/triage_crashes.sh results/out_stateful/default/crashes/id:000000...
```

If the payload is a true zero-day, AddressSanitizer will immediately halt execution and print a red stack trace pointing to the vulnerable C++ line.
