# USB 2.0 PHY UVM environment

This project retains the attached UART bench's folder structure. `apb_agent/` contains the USB SIE byte agent (the PHY has no APB port); `serial_agent/` contains the active USB line peer, `env/` contains top, scoreboard, wire oracle and coverage, `rtl/` contains supplied DUT plus interfaces, `test/` contains directed tests, `test_pkg/` contains the UVM package, and `sim/` contains Xcelium commands.

The peer drives received SYNC/NRZI/stuffed payload/EOP through the resolved D+/D- analog front-end pins and separately listens to DUT TX pins. The TX oracle constructs symbols from accepted SIE handshakes, compares line symbols and duration, decodes bytes independently, then the scoreboard compares bytes and packet tags. On RX, expected bytes are published by the peer before it drives SYNC; the scoreboard checks actual DataOut and SOP/EOP and status events. See `SLAVE_MODEL_README.md` and `TESTPLAN.md`.

Read `ANALYSIS.md` for the full port contract, UART files excluded, design mismatches, AFE assumptions and implemented scope. Run from `sim/` with Cadence Xcelium and a UVM 1.2 compatible library (`make compile`, `make regression`). No suitable simulator is installed here: all test results remain Pending. HS receive is unsupported in the supplied CDR; HS transmit is included. The run scripts require this source tree layout.
