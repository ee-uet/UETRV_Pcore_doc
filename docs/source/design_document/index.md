# UETRV_Pcore: Design Document
UETRV_Pcore is a RISC-V based application class SoC integrating a 5-stage pipelined processor with memory and peripherals. Currently, the core implements RV32IMAZicsr ISA based on User-level ISA Version 2.0 and Privileged Architecture Version 1.11 supporting M/S/U modes. Following are the key features of the SoC:

## Introduction

### Key Features
- 32-bit RISC-V ISA core that supports base integer (I) and multiplication and division (M), atomic (A) and Zicsr (Z) extensions (RV32IMAZicsr).
- Supports user, supervisor and machine mode privilege levels.
- Support for instruction / data (writeback) caches.
- Sv32 based MMU support and is capable of running Linux.
- Cache size, TLB entries etc., are configurable.
- Intergated PLIC, CLINT, uart, spi peripherals. 
- Uses RISOF framework to run architecture compatibility tests.

### System Design Overview
The UETRV_Pcore is an applicaion class processor capable of running Linux. A simplified 5-stage pipelined block diagram is shown below. The M-extension is implemented as a coprocessor while memory-management-unit (MMU) module is shared by instruction and data memory (alternatively called load-store-unit (LSU)) interfaces of the pipeline. Specifically, the page-table-walker (PTW) of the MMU is shared and there are separate TLBs (translation look aside buffers) for instruction and data memory interfaces. The A-extension is implemented as part of the LSU module.

![pipeline](../images/pipeline.png)

The SoC block diagram shows the connectivity of the core with memory sub-system as well as different peripherals using data bus. The boot memory is connected to both instruction and data buses of the core using a bus multiplexer. The instruction and data caches share the main memory using a bus arbiter. Different necessary peripherals are connected using the data bus. 

![soc](../images/soc.png)

### SoC Memory Map
The memory map for the SOC is provided in the following table.
| Base Address        |    Description            |   Attributes    |
|:-------------------:|:-------------------------:|:---------------:|
| 0x8000_0000         |      Memory               |      R-X-W      |
| 0x9000_0000         |      UART                 |      R-W        |
| 0x9400_0000         |      PLIC                 |      R-W        |
| 0x9C00_0000         |      SPI                  |      R-W        |
| 0x0200_0000         |      CLINT                |      R-W        |
| 0x0000_1000         |      Boot Memory          |      R-X        |

- `R: Read access`
- `W: Write access`
- `X: Execute access`

## Design Details
The core and SoC design details are outlined in the following subsections.
- Instruction fetch and decode stages
- Execute stage and M-extension
- Memory and writeback stages
- Pipeline controller
- Memory management unit (MMU) details
- Booting and peripherals

### Instruction Fetch and Decode Stages
The instruction fetch stage of the pipeline reads instructions either from the boot memory (`bmem`) or from the instruction cache (`icache`). The default/reset value of the program counter (PC) starts execution from `bmem`, which normally contains the zero-order bootloader. After booting, the PC jumps to the main memory region to start user program execution. 

If virtualization is turned on then both virtual-to-physical address translation (using memory management unit (MMU)) as well as fetching the instructions from `icache` are performed during fetch stage. Cascading the two key operation can easily make this phase to become crtical path. An instruction page fault exception from the MMU (due to failuare of virtual address translation) enters the pipeline at fetch stage.    

The decode stage performs instruction decoding as well as operand fetching from the register-file. The register file write operation is performed on the falling edge of the clock to reduce the hardware cost of full forwarding.  

### Execute Stage and M-extension
Execute stage implements all the integer ALU operations required by the base RV32I. The forwarding multiplexer is implemented in the execute stage. The input operands to the M-extension are provided by the execute stage, while the result from M-extension is provided to the memory stage. M-extension uses a multi-cycle implementation.  

### Memory and Writeback Stages
Memory stage houses load-store-unit (LSU), control-status-register (CSR) register-file and A-extension. LSU module performs load/store operaions from/to dcache as well as peripheral devices, using data bus. The load/store operations from/to peripheral devices are non-cacheable. The LSU module does not support misaligned accesses. When virtualization is enabled LSU module interacts with the MMU module using dedicated request response channel, for address translation. Further details regarding virtual address translation are provided later in MMU module related subsection.

#### A-extension  
The execution of A-extension instructions is a multi-cycle operation and leads to pipeline stall. There are two groups of instructions in A-extension. First group comprise of atomic-memory-operation (AMO) instructions while second group involves load-reserved (LR) and store-conditional (SC) instructions.

The AMO instructions involve the following three phases:
- Loading one operand from memory
- Performing the (requested) operation
- Storing the result to memory (at the same address used by load operation in the first phase)

The LR and SC instructions work as a pair. When an LR instruction is executed it requires temporary storage for holding both the address as well as the data. For that purpose we have implemented a **single entry buffer** that becomes occupied on executing an LR instruction. The buffer remains occupied until a corresponding SC instruction is executed. The buffer is freed irrespective of the fact that SC instruction execution was successful. Using a single entry buffer does not limit the performance, assuming nested LR instructions are not permitted.    

#### CSR Module
The memory stage also implements the CSR read/write operations including exception/interrupt handling. External interrupts are asynchronous and require synchronization for  their precise handling. An approach based on interrupt continuable instruction is followed for percise interrupt handling. For that purpose, the interrupt is put on hold if the pipeline was either stalled or being flushed at the time of occurrence of the interrupt.    

### Pipeline Controller
Key operations performed by the pipeline controller involve forwarding/stalling/flushing of the pipeline stages. The current version implements full forwarding. LSU operations lead to one stall cycle. In case of load use hazard, forwarding is always performed from the writeback stage and results in an extra stall cycle. Below are some possible scenarios leading to pipeline stalls.
- Fetch/decode/execute/memory stages stall due to data TLB miss, dcache miss, executing atomic and other A-extension instructions
- Fetch/decode/execute/memory stages stall due to M-extension operation 
- Fetch stage stall due to icache miss 

Pipeline is flushed due to:
- Execution of jump or conditional-branch instruction
- Exception or interrupt handling

Execution of a jump or conditional-branch instruction leads to flushing fetch and decode stages of the pipeline. In case of an exception or interrupt fetch, decode and execute stages of the pipeline are flushed.

### MMU Details
The memory management unit (MMU) is responsible for address translation when activated by configuring the corresponding CSR (i.e. **satp** register) and ensuring that the current privilege mode is lower than the machine mode. Enabling address translation also requires the ability to handle page faults. The MMU implements page-based 32-bit virtual-memory system conforming to Sv32 specifications. The MMU module interfaces with the LSU and fetch modules are shown in the accompanying figure below.

![mmu](../images/mmu.png)

The MMU module implements separate TLBs for instruction and data memory interfaces along with a shared hardware page table walker (PTW). It is possible that we encounter both ITLB as well as DTLB misses during the same cycle. In that case, the hardware PTW arbitrates the address translation requests and prioritises DTLB miss over ITLB miss. 

### Booting and Peripherals
On processor reset, the first instruction executed depends on the PC reset value defined by the user configuration parameter `PC_RESET` defined in `config_parameters.svh` in the `rtl\defines`. The default value of the parameter `PC_RESET` is `0x00001000`, which is the starting address of the boot memory. The boot memroy is read only and is preinitialized with the zero-level boot loader (ZLBL). The ZLBL usually contains early system initializtions including some interface intialization required for loading either next level boot loader or user application. The boot memory can be accessed either from instruction or from data bus interface. Since boot memory is single port memory, a bus multiplexer is used to interface it with both instruction and data bus interfaces.   

Curerntly the following peripherals are integrated with the processor core using the data bus interface. All these interfaces are non-cacheable and their address map is available in the [`SoC Memory Map`](###SoC) subsection.  
- PLIC (platform level interrupt controller)
- CLINT (core level interruptor)
- UART
- SPI
