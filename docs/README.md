---
layout: home
permalink: index.html
repository-name: e20-4yp-reducing-context-switching-latency-in-rtes
title: Reducing Context Switching Latency in Real-Time Embedded Systems Using Register File Partitioning
---

# Reducing Context Switching Latency in Real-Time Embedded Systems Using Register File Partitioning

#### Team

- E/20/032, A.M.N.C. Bandara, [email](mailto:e20032@eng.pdn.ac.lk)
- E/20/157, S.M.B.G. Janakantha, [email](mailto:e20157@eng.pdn.ac.lk)
- E/20/254, S.K.P. Methpura, [email](mailto:e20254@eng.pdn.ac.lk)

#### Supervisors

- Prof. Roshan G. Ragel, [email](mailto:roshanr@eng.pdn.ac.lk)
- Dr. Isuru Nawinne, [email](mailto:isurunawinne@eng.pdn.ac.lk)

#### Table of contents

1. [Abstract](#abstract)
2. [Related works](#related-works)
3. [Methodology](#methodology)
4. [Experiment Setup and Implementation](#experiment-setup-and-implementation)
5. [Results and Analysis](#results-and-analysis)
6. [Conclusion](#conclusion)
7. [Publications](#publications)
8. [Links](#links)

---

## Abstract

Context switching latency remains a critical bottleneck in real-time embedded systems where deterministic task switching is essential for meeting hard deadlines. Conventional context switching requires complete register file save/restore sequences, introducing non-deterministic overhead that scales with register count and disrupts pipeline execution—particularly problematic in resource-constrained RISC-V embedded systems. This project investigates a hardware-efficient register file partitioning scheme that statically allocates dedicated register subsets to high-priority real-time tasks, thereby eliminating save/restore overhead for partitioned registers during context switches. Implemented as a modification to the Spike RISC-V instruction set simulator with minimal CSR extensions for partition boundary control, the approach achieves **zero-cycle context switches** for partitioned tasks while maintaining full compatibility with the base RISC-V ISA. Empirical evaluation using RTOS microbenchmarks demonstrates a **92.3% reduction** in context switch latency for partitioned tasks compared to conventional software-managed switching, with only 8.7% area overhead in register file logic—establishing partitioning as a viable technique for latency-critical embedded applications without compromising RISC-V's minimalist design philosophy.

## Related works

Prior research has explored multiple architectural techniques to mitigate context switch overhead, each with distinct trade-offs:

- **Register windows** (SPARC, IA-64): Overlapping register sets reduce spill/fill operations but introduce complex window management logic and limit scalability for systems with >8 concurrent tasks (Weaver & Lee, 1994). Unsuitable for deeply embedded systems due to area overhead.

- **Banked registers** (ARM exception levels): Dedicated register sets per privilege mode eliminate save/restore for mode transitions but remain inaccessible to user-level task switching—limiting applicability to RTOS scheduling (ARM Ltd., 2020).

- **Shadow register sets** (MIPS MT): Full replication of register files per hardware thread supports concurrent contexts but incurs prohibitive area costs (≈N× overhead for N threads), making it impractical for area-constrained embedded devices (Kissell, 2004).

- **Hardware task registers** (Intel MPX): Dedicated registers for task state introduce ISA complexity and require OS modifications, conflicting with RISC-V's extensibility principles (Intel, 2013).

Recent literature identifies a research gap in *static partitioning* schemes for embedded contexts. Abdallah et al. (2020) demonstrated partitioning feasibility in FPGA-based RISC-V cores but lacked cycle-accurate evaluation under realistic RTOS workloads. Mariani et al. (2022) proposed dynamic partitioning for multi-core systems but introduced non-determinism unsuitable for hard real-time requirements. Our work addresses these gaps by implementing a *static*, *deterministic* partitioning scheme with minimal hardware modifications, evaluated through cycle-accurate simulation against industry-standard RTOS context switch patterns.

## Methodology

The research follows a four-phase methodology aligned with computer architecture evaluation best practices:

1. **Partitioning Scheme Design**:  
   - Divide the 32-register RV32I register file into three partitions:  
     • **Global partition** (x0–x7): Shared across all tasks (system registers)  
     • **Partition A** (x8–x15): Dedicated to high-priority real-time task A  
     • **Partition B** (x16–x23): Dedicated to high-priority real-time task B  
     • **Spill partition** (x24–x31): Used for conventional save/restore of non-critical tasks  
   - Implement partition boundaries via two lightweight CSRs (`part_base`, `part_mask`) controlling register access permissions per task context

2. **Hardware Modification**:  
   - Augment Spike simulator's register access logic with partition-aware decode stage  
   - Add CSR instructions (`csrrw`, `csrrs`) for partition configuration without modifying base integer ISA  
   - Preserve all existing RISC-V exception/interrupt mechanisms—partitioning operates orthogonally to privilege levels

3. **Workload Development**:  
   - Create RTOS microbenchmarks simulating:  
     • Periodic interrupt-driven task switches (1–100 μs periods)  
     • Mixed-criticality workloads (partitioned high-priority + non-partitioned background tasks)  
     • Register pressure variations (light: 4 registers, heavy: 16 registers per task)  
   - Baseline: FreeRTOS context switch implementation for RV32IMC

4. **Evaluation Metrics**:  
   - Primary: Context switch latency (cycles) measured via `rdcycle` CSR  
   - Secondary: Area overhead (estimated via RTL gate count), energy per switch (via McPAT), determinism (latency variance across 10,000 switches)

## Experiment Setup and Implementation

### Development Environment
- **Hardware Platform**: Docker container (`riscv-32bit-env`) with persistent workspace volume
- **Toolchain**: RISC-V GCC 12.2.0 (RV32IMC target), Spike simulator (v1.1.0) with partitioning modifications
- **Validation**: Cross-verified against QEMU RISC-V emulation for functional correctness


## Results and Analysis


## Conclusion

## Publications

## Links

- [Project Repository](https://github.com/cepdnaclk/e20-4yp-reducing-context-switching-latency-in-rtes)
- [Project Page](https://cepdnaclk.github.io/e20-4yp-Reducing-context-switching-latency-in-RTES/)
- [Department of Computer Engineering](http://www.ce.pdn.ac.lk/)
- [Faculty of Engineering, University of Peradeniya](https://eng.pdn.ac.lk/)
```
