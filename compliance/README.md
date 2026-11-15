# RISC-V compliance results (RV32I + M)

**Result: 47 / 47 tests pass** (39 RV32I, 8 RV32M), checked against the Sail RISC-V reference model.

## Setup tested

| Item | Version |
|---|---|
| Test suite | riscv-arch-test, branch `act4`, commit `1c3e9f2a8` |
| Reference model | sail-riscv 0.16 (`sail_riscv_sim`) |
| Compiler | xPack `riscv-none-elf-gcc` 15.2.0-1 |
| Simulator | Verilator 5.032 |
| Environment | Ubuntu on WSL |

Core configuration under test: RV32I + M (with Zmmul) and no CSRs or traps.

## How it works

1. ACT4 compiles each test into an ELF, and Sail runs it to produce a reference signature.
2. `sim_main.cpp` (Verilator) loads the same ELF into the core's instruction and data memories, both based at `0x80000000`, and runs the core until the test writes a non-zero value to `tohost` (2,000,000-cycle timeout).
3. It dumps the signature region (`begin_signature` to `end_signature`) from data memory.
4. `run_tests.sh` compares that signature against Sail's, byte for byte. A test passes only if they are identical.

The repository's modules had no top level, so `rtl/rv32i_soc_top.sv` connects them: it instantiates `rv32i_if_id_ex`, `rv32i_mem_stage` and `rv32i_data_memory`, adds `rv32i_inst_memory` (combinational read), and derives register-write, load/store and write-back control from the opcode. The existing core modules are unchanged. This result covers those modules plus that wrapper.

## Files

| Path | Purpose |
|---|---|
| `rtl/rv32i_soc_top.sv` | Wrapper that connects the core modules and the memories |
| `rtl/rv32i_inst_memory.sv` | 256 KiB byte-addressable instruction memory |
| `compliance/sim_main.cpp` | Verilator testbench: ELF loader, run loop, signature dump |
| `compliance/run_tests.sh` | Runs every test and compares signatures |
| `compliance/config/` | ACT4 DUT config: `test_config.yaml`, `rvmodel_macros.h`, `link.ld`, `mycore-RVI20U32.yaml`, `sail.json` |

Config summary: the Sail RVI20U32 config trimmed to I, M, Zmmul and Sm. A, F, D, Zicntr, Zihpm, Zifencei, Zaamo, Zalrsc, Zca, Zcf and Zcd are disabled in `sail.json`, and mstatus FS/VS are off. `rvmodel_macros.h` halts by storing to `tohost` and has no UART or CLINT.

## Reproduce

The script expects this layout under `~/compliance_test`. Install Sail 0.16, the xPack GCC, Verilator 5.x, `uv`, `mise` and Ruby with bundler first.

    mkdir -p ~/compliance_test/design ~/compliance_test/sim
    cd ~/compliance_test
    git clone https://github.com/riscv/riscv-arch-test.git
    cd riscv-arch-test && git checkout act4 && mise trust

    # point the stock Sail config at the xPack toolchain
    sed -i 's/riscv64-unknown-elf-/riscv-none-elf-/' config/sail/sail-RVI20U32/test_config.yaml

    # golden signatures
    UV_PYTHON=3.12 make CONFIG_FILES=config/sail/sail-RVI20U32/test_config.yaml \
      EXTENSIONS="I,M" JOBS=1

    # DUT config and ELFs
    mkdir -p config/mycore/mycore-RVI20U32
    cp <repo>/compliance/config/* config/mycore/mycore-RVI20U32/
    make CONFIG_FILES=config/mycore/mycore-RVI20U32/test_config.yaml \
      EXTENSIONS="I,M" elfs

    # simulator
    cp <repo>/rtl/*.sv ~/compliance_test/design/
    cp <repo>/compliance/sim_main.cpp <repo>/compliance/run_tests.sh ~/compliance_test/sim/
    cd ~/compliance_test/sim
    verilator --cc --exe --build -Wno-fatal --public-flat-rw --top-module rv32i_soc_top \
      --Mdir obj_dir -O2 -I$HOME/compliance_test/design \
      $HOME/compliance_test/design/*.sv sim_main.cpp

    # run
    ./run_tests.sh

`--public-flat-rw` is required because the top module has no outputs, so Verilator would otherwise remove the memories as dead logic.

## Scope and limitations

- One test per instruction (the `-00` variants), so this is a conformance gate and not exhaustive verification.
- No Zicsr, traps or privileged-mode tests. The suite was configured for a core without them.
- Misaligned-access behavior is not tested.
- Instruction and data memories are separate and both preloaded with the program image, so self-modifying code is not supported.
