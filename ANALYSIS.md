# Source analysis and interface contract

## Sources examined

- `uart_verification(1).zip`: active APB/SIE role, serial peer role, environment, scoreboard, tests, package and simulation layout. Omitted `env/uartsb_old.sv` (superseded duplicate), commented `serial_agent/serial_seq.sv` and `apb_agent/apb_pkg.sv` (not active), commented assertion file, UART RTL and generated `INCA_libs`/SimVision/coverage caches (unrelated or generated).
- `USB_2_0_PHY_Finaltop_Micro-Architecture_Specification(2).docx` and `USB_2_0_CDR_Micro-Architecture_v6.docx`: top integration, CDR timing and supported LS/FS receive.
- USB 2.0 specification: line J/K/SE0/SE1, FS/LS and HS SYNC, NRZI, bit stuffing and EOP conventions. The executable oracle is limited to features implemented in the supplied RTL.
- `rtl/*.sv`: provided top and dependencies, copied as supplied; the testbench binds to the implemented `phy_top` ports. Testbench interfaces were added to `rtl/` to retain the UART layout.

## `phy_top` port contract

All SIE signals are synchronous to the 480 MHz `clk`; `rst_n` is active low. The LS and FS CDR uses 320 and 40 system clocks per bit; HS TX uses one clock per bit. Output reset and idle behavior below is checked where meaningful; data outputs need not have meaningful idle payload.

| Ports | Direction | Meaning / reset or idle |
| :--- | :--- | :--- |
| `clk`, `rst_n`, `usb_speed` | in | 480 MHz, active-low reset, LS/FS/HS; reserved speed maps to FS in top |
| `sie_data[WIDTH-1:0]`, `sie_valid`, `sie_sop`, `sie_eop`, `sof_packet` | in | SIE byte and tags; accepted on `sie_valid && sie_ready`; driver drives zero during initial reset |
| `sie_ready` | out | FIFO backpressure; do not infer a fixed reset value |
| `dp`, `dm`, `tx_valid` | out | Local TX line and ownership; TX idle J at FS/LS, SE0 at HS per supplied TX RTL |
| `analog_d_plus`, `analog_d_minus`, `analog_serial`, `analog_enable` | in | External receive AFE levels, one-bit serial J/K representation, CDR enable; peer drives J=1 and K=0, disables receiver during local TX |
| `DataOut[WIDTH-1:0]`, `RxValid`, `RxActive`, `RxError` | out | SIE receive byte and status; valid/active/error deassert in reset |
| `LineState[1:0]` | out | Actual RTL mapping `{d_plus,d_minus}`; header comment differs and is not the oracle |
| `rx_sop`, `rx_eop` | out | First/last byte tags, valid only with `RxValid`; deassert in reset |
| `rx_stuff_err`, `rx_align_err`, `rx_line_err`, `rx_sync_err`, `rx_bus_reset` | out | RX status; all deassert in reset; bus reset LS/FS only |

## Architecture and RTL discrepancies / assumptions

1. The final-top document describes HS downstream receive, but `usb2_cdr_top` explicitly parks its lock/strobe at HS; `phy_top` has no external HS RX bypass. The user confirmed HS RX is out of scope. HS TX is exercised.
2. `phy_rx_top` comment says `LineState={DM,DP}`, but code assigns `{d_plus,d_minus}`. The requested testbench follows actual RTL order.
3. CDR receives 480 MHz independent of `CLK_HZ` top parameter; use default 480 MHz. WIDTH=8 and default RX/TX parameters are the configured run; other parameters need separate elaboration and tests.
4. `analog_serial` is independent of D+/D- at the port. The peer supplies the NRZI line level J=1, K=0 with consistent line pins. Confirm the actual external AFE mapping before silicon-signoff.
5. The top exposes no line arbitration/turnaround output. Testbench resolves two drive enables and gates analog receiver on local `tx_valid`; this external squelch is a testbench AFE assumption. Simultaneous drive is an error.
6. Only PHY byte transport is checked. PID, token address/endpoint, CRC, transaction retries and host scheduling are SIE functions, not implemented at this interface.
7. Timed CDR ±1 system-clock alternating jitter is directed functional stimulus, not a rate offset, jitter mask, deglitch threshold or compliance measurement. Such characterization remains additional work.
8. Default `ALIGN_IN_RXERROR=0` is assumed by the pin-level RxError checker; nondefault variants need checker parameterization.

No simulation results are claimed. A UVM simulator with covergroup support is required.
