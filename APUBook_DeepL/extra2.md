# OAM DMA Circuit

![oam_dma_parts](/APUBook/imgstore/oam_dma_parts.png)

OAM DMA is one of the two DMA controllers inside the APU.

## Signals

- ACLK and #ACLK: APU Clock.
- PHI1: The first half cycle of the CPU
- R/W: CPU R/W signal (operating mode; 1: Read Mode, 0: Write Mode)
- RUNDMC: OAM DMA lockout signal from DPCM DMA side. Used to suspend OAM DMA when two DMAs intersect
- SPRS: A signal that enables OAM counter (low 8 bits of the address). The signal is silenced by RUNDMC.
- SPRE: OAM counter overflow signal. Used to determine if an OAM DMA is complete.
- DMCReady: The DPCM DMA ready signal. Combined with the OAM DMA ready signal (oamdma_rdy) to produce the RDY signal for the 6502
- SPR_CPU: Perform a read byte in the DMA Buffer at the OAM Counter address. The signal is silenced by RUNDMC.
- SPR_PPU: Write the value in the DMA Buffer to register $2004. The signal is silenced by RUNDMC.

## Main circuit components

**RS-FF to store a write event to register $4014 (signal W4014)**

This RS-FF is set when write operation W4014 is activated. The W4014 signal, like the other RegOps, is active only during PHI2 (second half of the CPU cycle).
This RS-FF is cleared by the RES signal and also immediately after the start of OAM DMA (when the NOSPR signal becomes 0) ("autocleared")

**CPU Read Cycle Detector**

Made simple: `read_cycle = (R_W == 1) & (PHI1 == 0)`
The need for the Read Cycle detector is due to the fact that the CPU ignores a low RDY signal during write cycles.

OAM DMA start event is considered to have occurred if:
- A write event has occurred to register $4014.
- Processor went into Read Cycle.

**RS-FF "DMA Enabler"**

Latches the OAM DMA start event (DOSPR signal). Cleared by SPRE signal and during reset (RES).

**"No Sprite DMA" latch (nospr_latch)**

This latch remembers the last DMA Enabler value, but only opens during #ACLK. Opening this latch during #ACLK is exactly related to the case of "unaligned DMA": even if all OAM DMA start events have been detected and stored on the DMA Enabler - the actual OAM DMA will not start until #ACLK has occurred.

**RS-FF DirToggle**

A regular FF that constantly changes its value from 0 to 1 and back, thus determining the direction of the OAM DMA.

**RDY Generator**

The RDY signal for the processor is simply the logical AND of signals oamdma_rdy and DMCReady.

## DMA Process

The NOSPR signal is the main driving force behind all OAM DMA. When NOSPR is 0 - the OAM DMA circuitry performs its activities to provide the OAM DMA process.

The OAM DMA process consists of the following activities:
- Alternating DMA direction change (DMA DirToggle), which eventually generates the SPR_PPU and SPR_CPU signals
- Activating the SPRS signal to turn on the OAM Counter (so that the OAM DMA address keeps increasing).

These processes are "silenced" by the RUNDMC signal.

OAM DMA Start:
![oam_dma_start](/APUBook/imgstore/oam_dma_start.png)

OAM DMA End:
![oam_dma_last](/APUBook/imgstore/oam_dma_last.png)

## DPCM DMA and OAM DMA Interaction

DPCM DMA affects the operation of OAM DMA in only two ways:
- The RUNDMC signal "silences" (nulls) the OAM Counter counting signal (SPRS) and the signals for OAM DMA direction (SPR_CPU/SPR_PPU). Thus, only the OAM Counter is actually suspended. DirToggle continues to work, so you need to maintain precise timing of the RUNDMC signal, so that at the time of RUNDMC "push-back" - DirToggle took the same value as before;
- DMCReady signal is simply "forwarded" to the RDY terminal of the processor. Of course, if oamdma_rdy signal is 0, then DMCReady signal has no effect. This can happen, for example, when DPCM DMA terminates "inside" OAM DMA: DPCM DMA will say DMCReady=1, but processor will not wake up anyway, because oamdma_rdy signal is still 0 (OAM DMA has not finished)
