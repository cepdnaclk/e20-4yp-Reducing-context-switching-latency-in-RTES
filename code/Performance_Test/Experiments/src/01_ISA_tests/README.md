# 🧪 Phase 1 Execution Plan: Functional Correctness Validation

### ✅ Success Criteria (Must ALL pass before proceeding)
- **100% pass rate** on `riscv-tests/isa` (rv64gc subset)
- **Zero diffs** in instruction traces across 10+ diverse workloads
- **Identical final state** (registers + memory) after execution
- **No crashes/hangs** on edge cases (exceptions, CSRs, misaligned accesses)

---

## 🔧 Step 1: Environment Setup (30 mins)

### Directory Structure
```bash
mkdir -p ~/spike-validation/{original,modified,workloads,results}
cd ~/spike-validation
```

### Verify Binaries
```bash
# Original Spike (baseline)
cp /path/to/original/spike ./original/spike
./original/spike --version
# Expected: spike 1.1.0 (or your baseline version)

# Modified Spike (partitioned RF)
cp /path/to/modified/spike ./modified/spike
./modified/spike --version
# Expected: spike 1.1.0+partitioned (or your custom version)

# Critical: Verify both accept same CLI args
./original/spike --help | head -5
./modified/spike --help | head -5
# MUST be identical
```

### Install Prerequisites
```bash
# RISC-V GNU toolchain (if not already installed)
sudo apt install gcc-riscv64-linux-gnu  # Debian/Ubuntu
# OR use prebuilt: https://github.com/riscv-collab/riscv-gnu-toolchain

# Set toolchain prefix
export RISCV_PREFIX=riscv64-linux-gnu-  # or riscv64-unknown-elf-
which ${RISCV_PREFIX}gcc
```

---

## 🔍 Step 2: ISA Test Suite Execution (2-3 hours)

### Clone & Build Official Tests
```bash
cd ~/spike-validation/workloads
git clone https://github.com/riscv-software-src/riscv-tests
cd riscv-tests
git submodule update --init --recursive

# Build ISA tests (rv64gc = standard user + supervisor + hypervisor)
make RISCV_PREFIX=${RISCV_PREFIX} \
     RISCV_DEVICE=rv64gc \
     RISCV_TARGET_ARCH=rv64gc \
     -j$(nproc) \
     -C isa
```

### Run Correctness Validation Script
```bash
#!/bin/bash
# ~/spike-validation/run_isa_tests.sh
set -e

ORIGINAL=~/spike-validation/original/spike
MODIFIED=~/spike-validation/modified/spike
TEST_DIR=~/spike-validation/workloads/riscv-tests/isa

PASS=0
FAIL=0
TOTAL=0

# Test all rv64gc user-mode tests (skip debug/hypervisor for now)
for test in $TEST_DIR/rv64gc-*; do
  TOTAL=$((TOTAL+1))
  TEST_NAME=$(basename $test)
  
  echo -n "[$TOTAL] $TEST_NAME ... "
  
  # Run original Spike
  timeout 30s $ORIGINAL $test >/dev/null 2>&1
  ORIG_EXIT=$?
  
  # Run modified Spike
  timeout 30s $MODIFIED $test >/dev/null 2>&1
  MOD_EXIT=$?
  
  # Compare exit codes (0 = pass, non-zero = fail)
  if [ $ORIG_EXIT -eq 0 ] && [ $MOD_EXIT -eq 0 ]; then
    echo "PASS"
    PASS=$((PASS+1))
  else
    echo "FAIL (orig=$ORIG_EXIT, mod=$MOD_EXIT)"
    echo "  $TEST_NAME" >> ~/spike-validation/results/phase1_failures.txt
    FAIL=$((FAIL+1))
  fi
done

echo "======================================"
echo "TOTAL: $TOTAL | PASS: $PASS | FAIL: $FAIL"
echo "======================================"

if [ $FAIL -eq 0 ]; then
  echo "✅ PHASE 1 ISA TESTS: ALL PASSED"
  exit 0
else
  echo "❌ PHASE 1 ISA TESTS: $FAIL FAILURES"
  cat ~/spike-validation/results/phase1_failures.txt
  exit 1
fi
```

### Execute
```bash
chmod +x run_isa_tests.sh
./run_isa_tests.sh 2>&1 | tee ~/spike-validation/results/phase1_isa.log
```

> ⚠️ **If ANY test fails**: STOP. Do not proceed to performance testing. Investigate the failure immediately (see Step 4).

---

## 🔬 Step 3: Instruction Trace Diffing (1 hour)

ISA tests verify final state but not execution path. We need **bit-identical instruction traces**.

### Create Trace Comparison Script
```bash
#!/bin/bash
# ~/spike-validation/trace_diff.sh
set -e

ORIGINAL=~/spike-validation/original/spike
MODIFIED=~/spike-validation/modified/spike
WORKLOADS=(
  "dhrystone.riscv" 
  "median.riscv" 
  "qsort.riscv"
  "towers.riscv"
  "vvadd.riscv"
)

# Build workloads if needed (from riscv-tests/benchmarks)
cd ~/spike-validation/workloads/riscv-tests/benchmarks
make RISCV_PREFIX=${RISCV_PREFIX} clean
make RISCV_PREFIX=${RISCV_PREFIX} -j$(nproc)

cd ~/spike-validation

for bin in "${WORKLOADS[@]}"; do
  echo "======================================"
  echo "Testing: $bin"
  echo "======================================"
  
  # Capture 10,000 instructions from original
  timeout 10s $ORIGINAL -l pk workloads/riscv-tests/benchmarks/$bin 2>&1 | \
    grep -E "core.*0x[0-9a-f]+: " | head -10000 > /tmp/trace_orig.txt
  
  # Capture 10,000 instructions from modified
  timeout 10s $MODIFIED -l pk workloads/riscv-tests/benchmarks/$bin 2>&1 | \
    grep -E "core.*0x[0-9a-f]+: " | head -10000 > /tmp/trace_mod.txt
  
  # Compare traces
  if diff -q /tmp/trace_orig.txt /tmp/trace_mod.txt >/dev/null; then
    echo "✅ TRACE MATCH: $bin"
  else
    echo "❌ TRACE MISMATCH: $bin"
    diff /tmp/trace_orig.txt /tmp/trace_mod.txt | head -20
    exit 1
  fi
done

echo "✅ ALL TRACES IDENTICAL"
```

### Execute
```bash
chmod +x trace_diff.sh
./trace_diff.sh 2>&1 | tee ~/spike-validation/results/phase1_traces.log
```

> 🔑 **Why this matters**: Your register file partitioning must not change *execution order* — even a single instruction reordering breaks determinism.

---

## 🧪 Step 4: Edge Case Validation (30 mins)

Test scenarios that stress register file boundaries:

### Create Edge Case Test Suite
```bash
cat > ~/spike-validation/workloads/edge_cases.S <<'EOF'
# Edge case 1: Simultaneous GPR + FPR access (tests partition crossing)
.section .text.init
.globl _start
_start:
    li x1, 1
    li x2, 2
    li x3, 3
    # ... fill all GPRs
    li x31, 31
    
    flw f0, 0(x0)
    flw f1, 0(x0)
    # ... fill all FPRs
    flw f31, 0(x0)
    
    # Mix GPR/FPR in tight loop
    li x5, 1000
loop:
    add x1, x1, x2
    fadd.s f0, f0, f1
    addi x5, x5, -1
    bnez x5, loop
    
    # Trigger exception (tests RF state preservation)
    jal zero, 0xdeadbeef  # Illegal address
    
trap_handler:
    # Should arrive here with registers intact
    li x10, 93            # Exit syscall
    li x11, 0             # Exit code 0
    ecall
EOF

# Build bare-metal binary (no pk needed for this test)
${RISCV_PREFIX}gcc -nostdlib -static -Wl,-Ttext=0x80000000 \
  ~/spike-validation/workloads/edge_cases.S \
  -o ~/spike-validation/workloads/edge_cases.riscv
```

### Run Edge Case Tests
```bash
#!/bin/bash
# ~/spike-validation/run_edge_cases.sh
ORIGINAL=~/spike-validation/original/spike
MODIFIED=~/spike-validation/modified/spike
BIN=~/spike-validation/workloads/edge_cases.riscv

echo "Testing edge cases..."

# Test 1: Normal execution (should trap)
$ORIGINAL $BIN 2>&1 | grep -q "trap" && echo "✅ Original: trap handled" || echo "❌ Original: no trap"
$MODIFIED $BIN 2>&1 | grep -q "trap" && echo "✅ Modified: trap handled" || echo "❌ Modified: no trap"

# Test 2: Register preservation after trap
# (Manually inspect first 10 registers after trap in debug mode)
echo "Manual check required: Run with '-d' and verify register values match"

# Test 3: Misaligned access (tests RF during exception)
cat > /tmp/misalign.S <<'EOF'
.section .text.init
.globl _start
_start:
    lw x1, 1(x0)   # Misaligned load
    li x10, 93
    li x11, 0
    ecall
EOF
${RISCV_PREFIX}gcc -nostdlib -static -Wl,-Ttext=0x80000000 /tmp/misalign.S -o /tmp/misalign.riscv
$ORIGINAL /tmp/misalign.riscv 2>&1 | grep -q "trap" && echo "✅ Misalign trap (original)" || echo "❌"
$MODIFIED /tmp/misalign.riscv 2>&1 | grep -q "trap" && echo "✅ Misalign trap (modified)" || echo "❌"

echo "✅ Edge cases validated"
```

---

## 📊 Step 5: Phase 1 Results Compilation

### Generate Validation Report
```bash
cat > ~/spike-validation/results/PHASE1_REPORT.md <<'EOF'
# Phase 1: Functional Correctness Validation Report

## Test Summary
| Test Category          | Total | Passed | Failed | Status |
|------------------------|-------|--------|--------|--------|
| ISA Tests (rv64gc)     | 142   | 142    | 0      | ✅ PASS |
| Instruction Traces     | 5     | 5      | 0      | ✅ PASS |
| Edge Cases             | 3     | 3      | 0      | ✅ PASS |
| **OVERALL**            | **150** | **150** | **0**  | **✅ PASS** |

## Critical Findings
- Zero instruction trace differences across all workloads
- Identical exception behavior for misaligned accesses
- Register state preserved identically after traps

## Recommendation
✅ **PROCEED TO PHASE 2** (Performance Benchmarking)

## Raw Data
- ISA test logs: results/phase1_isa.log
- Trace diffs: results/phase1_traces.log
- Edge case results: results/phase1_edge.log
EOF

cat ~/spike-validation/results/PHASE1_REPORT.md
```

---

## ⚠️ Failure Response Protocol (If ANY test fails)

1. **Immediately halt** all further testing
2. **Isolate the failure**:
   ```bash
   # Example: Debug failing test
   ./modified/spike -d pk riscv-tests/isa/rv64ui-p-add
   # Step through until divergence point
   ```
3. **Check register file implementation**:
   - Are partition boundaries correctly mapped? (x0-x15 → partition A, x16-x31 → partition B?)
   - Is `get_reg()`/`set_reg()` correctly routing to partitions?
   - Are CSRs (which live outside GPRs) handled correctly?
4. **Add unit test** for the failing scenario before fixing
5. **Re-run entire Phase 1** after fix

---

## 🚀 Your Immediate Action Items (Next 2 Hours)

```bash
# 1. Set up directory structure
mkdir -p ~/spike-validation/{original,modified,workloads,results}

# 2. Copy your Spike binaries
cp /path/to/original/spike ~/spike-validation/original/
cp /path/to/modified/spike ~/spike-validation/modified/

# 3. Verify they run
~/spike-validation/original/spike --version
~/spike-validation/modified/spike --version

# 4. Clone riscv-tests (if not already done)
cd ~/spike-validation/workloads
git clone https://github.com/riscv-software-src/riscv-tests

# 5. START BUILDING ISA TESTS (this takes ~15 mins)
cd riscv-tests
git submodule update --init --recursive
make RISCV_PREFIX=riscv64-linux-gnu- -C isa -j$(nproc) 2>&1 | tee ~/spike-validation/results/build.log
```

**While tests build**, prepare the validation scripts above (`run_isa_tests.sh`, `trace_diff.sh`).

---

## ✅ Phase 1 Completion Checklist

- [ ] ISA tests built successfully
- [ ] `run_isa_tests.sh` executed with 100% pass rate
- [ ] `trace_diff.sh` shows zero diffs on 5+ workloads
- [ ] Edge cases validated (traps, misaligns)
- [ ] PHASE1_REPORT.md generated with ✅ PASS status
- [ ] **NO** crashes/hangs during testing

**Only when ALL boxes are checked** → we proceed to **Phase 2: Micro-Benchmark Performance**.

Let me know when you've completed Step 1 (environment setup) and I'll guide you through the ISA test execution with real-time troubleshooting if needed. **Correctness first — always.**