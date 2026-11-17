# Initial Plan: FPGA Market-Data-to-Order Accelerator for HFT

Repository: https://github.com/wzm-540/ECE9463_Proj
Status: initial proposal; performance figures below are targets, not measured results.

## 1. Project Team

| Member | Email |
| --- | --- |
| Xianhui Yang | |
| Jiheng Shui | |
| Xuan Jia | |
| Ziming Wang | zw5112@nyu.edu |

## 2. IP Definition

**Purpose and scope.** We propose a SystemVerilog hardware IP for the trading-firm side of a high-frequency trading system. It consumes Nasdaq-style market data, maintains a bounded displayed order book, evaluates a deterministic buy strategy with pre-trade limits, and emits OUCH order-entry payloads. The objective is predictable market-update-to-order latency and exact protocol/state behavior. It is an educational accelerator, with no claim of profitable trading or exchange certification. A simulation-based implementation is the required outcome; board deployment is optional, consistent with the [course project scope](https://sdrangan.github.io/hwdesign/docs/projects/).

The minimum implementation handles one configured stock/stock-locate code, one MoldUDP64 session, and at most 32 active displayed orders for that stock. Replay starts from an empty book and includes every supported update in order. Capacity overflow invalidates the book rather than silently approximating the best price. This bounds storage and verification effort within four weeks; it does not represent an unrestricted live Nasdaq book.

**Host and peripheral interaction.** A processor system (PS), or a simulated host, configures the stock symbol/locate code, expected feed session/sequence, fair value, strategy thresholds, order size, risk limits, and OUCH template through a 32-bit AXI4-Lite slave. It can arm, disarm, clear state, and read status/counters. Configuration changes require disarming and draining the pipeline; multiword values become active only on a commit command. Reset starts disarmed.

Input is a 64-bit AXI4-Stream carrying complete UDP payloads containing MoldUDP64 packets, up to 1,500 payload bytes, with `TKEEP` identifying valid bytes and `TLAST` ending each datagram. An external adapter handles Ethernet MAC/PHY, IP/UDP checksums, and destination filtering; a testbench supplies equivalent payloads. Output is a separate 64-bit AXI4-Stream carrying one OUCH 4.2 Enter Order application message per frame. Network-byte order and fixed-width ASCII fields follow the [Nasdaq OUCH specification](https://www.nasdaqtrader.com/content/technicalsupport/specifications/tradingproducts/ouch4.2.pdf). A host/session adapter owns SoupBinTCP/TCP transport, login, acknowledgments, executions, and recovery. These functions and full NIC/PCIe integration are outside the minimum IP. AXI DMA and an AXI interconnect may provide board-side replay/capture and register access later.

**Defined operations.** MoldUDP64 framing uses its 20-byte header and two-byte message lengths. The expected sequence advances by the number of messages, including messages for other stocks. Heartbeats do not advance it; end-of-session disables trading. A gap, duplicate/out-of-order packet, session mismatch, malformed length, or oversized packet latches a fault. The host must disarm, clear, and replay a consistent stream before rearming; automatic retransmission is outside scope.

The ITCH 5.0 decoder supports Add (`A/F`), Execute (`E/C`), Cancel (`X`), Delete (`D`), Replace (`U`), System Event (`S`), and Stock Trading Action (`H`). It skips known non-book messages by validated length; unsupported book-changing events for the selected stock invalidate the book. Other stocks are filtered by locate code. A halt or session close inhibits orders. Trading starts only after a valid trading state and an explicit host arm.

Each book slot contains `(valid, reference_id[63:0], side, price[31:0], shares[31:0])`. Add inserts a unique reference; Execute/Cancel subtract shares and invalidate zero quantities; Delete invalidates the slot; Replace atomically changes reference, price, and shares while preserving side. Execution-with-price reduces the resting quantity without changing the remaining order's display price. Unknown references, duplicate references, quantity underflow, and insufficient capacity latch a fault. State updates commit in feed order.

After each committed book update, compute:

```text
best_bid = max(price_i for valid buy slots)
best_ask = min(price_i for valid sell slots)
ask_size = sum(shares_i for valid sell slots with price_i == best_ask)
spread   = best_ask - best_bid
edge     = fair_value - best_ask
```

The quote is valid only if both sides exist and `best_bid < best_ask`. All prices are unsigned integers in units of $0.0001, matching ITCH Price(4) and OUCH 4.2 prices. Differences use widened signed arithmetic; aggregate quantity uses 37 bits for 32 slots. No floating point or division is required.

For each update, a candidate buys fixed quantity `q` at limit price `best_ask` iff the quote is valid, `edge >= edge_min`, `spread <= spread_max`, `ask_size >= q`, and the configured cooldown has elapsed. The risk gate additionally requires arm/trading status, no fault, `0 < q < 1,000,000`, `q <= per_order_limit`, `best_ask <= price_cap`, and `reserved_shares + q <= run_share_budget`. On accepting a candidate into a reserved output FIFO slot, increment reserved shares and allocate a unique 14-character token from a host-provided day-unique prefix and monotonic counter. Counter exhaustion inhibits orders. Rejected candidates do not change these states. One update produces at most one order.

The run budget conservatively counts all emitted/pending buys as potential fills and is never refunded during a run. The host owns actual position accounting and may reset the budget only after disarming, draining output, and reconciling outstanding orders. This makes the minimum design buy-only with bounded exposure. The template fixes remaining OUCH fields, including immediate-or-cancel time-in-force, and the encoder fills symbol, token, side, shares, and price. Fault/disarm blocks new reservations; an already accepted message finishes so that the output frame is never truncated.

**Why hardware.** Independent reference comparisons, min/max reductions, field extraction, and integer threshold checks map to parallel comparators and pipelined logic. Keeping book state and the decision path on chip removes a processor round trip from the per-update path. Fixed capacity makes latency and resource costs measurable. The supplied HLS and RTL HFT papers motivate this streaming partition; their sub-microsecond results are prior work, not predictions for our smaller prototype.

## 3. IP Architecture

```text
UDP-payload AXI stream
  -> packet buffer / MoldUDP64 validator
  -> ITCH decoder -> book update engine -> quote reducer
  -> strategy -> risk gate / token allocator -> order FIFO
  -> OUCH encoder -> application-payload AXI stream
                  ^
PS AXI4-Lite -> configuration/status controller
```

| Sub-module | Responsibility and interface |
| --- | --- |
| Packet buffer and MoldUDP64 validator | Buffer and validate the complete datagram before releasing any contained messages. Check header/session/sequence/count/lengths. Emit ordered ITCH message bytes plus sequence metadata; malformed packets produce no book updates. |
| ITCH decoder | Decode fixed offsets/endian fields and apply stock filtering. Emit a ready/valid event record containing operation, old/new reference, side, price, quantity, sequence, and trading-state flags. |
| Book update engine | Own the 32-slot register array. Parallel ID comparisons find the target/free slot; a controller commits one event at a time. Emit a stable post-update snapshot and sequence tag. |
| Quote reducer | Use pipelined min/max comparator trees and a quantity adder tree over the snapshot. Emit best bid/ask, best-ask size, validity, and sequence. The update engine waits until the snapshot is consumed, avoiding read/write hazards. |
| Strategy | Apply widened integer arithmetic, thresholds, and cooldown. Emit a candidate record `{sequence, symbol, buy, quantity, limit_price}` or count a no-trade decision. |
| Risk gate and token allocator | Check limits and atomically reserve budget, token, and output capacity. Emit an approved-order record; record rejection reasons. A final arm/fault check prevents queued candidates from bypassing a later inhibit. |
| OUCH encoder | Serialize approved records and configured template into protocol bytes, preserving packet boundaries under backpressure. It never modifies market-book state. |
| Configuration/status controller | Own AXI4-Lite registers, shadow/active configuration, arm/clear controls, sticky faults, counters, and cycle timestamps. Distribute stable settings and a global inhibit. |

Internal records travel through bounded synchronous FIFOs using `valid/ready`; a transfer occurs only when both are asserted. All stream data/metadata remain stable while stalled. The core initially uses one clock with synchronous reset; no clock-domain crossing is required. The packet buffer and FIFOs use inferred RAM, while the small order table uses registers to permit parallel lookup. Backpressure propagates to the replay source without dropping events. A future non-stallable network adapter must provide buffering and explicitly invalidate the feed on overflow.

**Incremental development and evaluation.** Decoder, book, arithmetic strategy, and encoder boundaries permit independent comparison against a Python golden model before integration. Planned ownership is Xianhui Yang: framing/ITCH; Jiheng Shui: book/quote; Xuan Jia: strategy/risk; Ziming Wang: host registers/OUCH/integration. All members review shared record formats and end-to-end tests. Week 1 freezes interfaces and golden vectors; week 2 implements and checks modules; week 3 integrates randomized streams/backpressure; week 4 performs synthesis, measurements, and documentation.

The planned Cocotb testbench compares every book update, decision, and encoded byte with the reference model. Directed cases cover partial/full executions, replace/delete, equal-price aggregation, overflow/unknown references, malformed datagrams, sequence/session faults, halts, risk boundaries, token exhaustion, reset, and stalled output. Reproducible synthetic fixtures avoid reliance on licensed historical data. Assertions check stable stalled streams, event ordering, no unauthorized order reservations, and exact output accounting after draining.

We target 156.25 MHz synthesis on an AMD FPGA part selected with the instructor; report the actual part/tool versions and achieved timing. The initial latency goal is at most 128 core cycles (819.2 ns at that clock) from acceptance of the triggering datagram's final beat to acceptance of its order's final output beat, with no prior backlog or output stalls. Also report first-input-beat-to-final-output latency, per-stage cycles, sustained accepted events/s, burst behavior, LUT/FF/BRAM/DSP use, and comparison against the same software model. This is a core-only target excluding network/session transport, not a 10 Gb/s line-rate or wire-to-wire claim. Failure to meet a timing target will be reported with the measured bottleneck.

**Reference basis.** Protocol layouts and update semantics use the supplied *Nasdaq TotalView-ITCH 5.0 Interface Specification* (March 6, 2015) and *MoldUDP64 Protocol Specification*. Architectural context also uses the supplied papers by Boutros et al., *Build Fast, Trade Fast: FPGA-based High-Frequency Trading using High-Level Synthesis*, and Kao et al., *An FPGA-Based High-Frequency Trading System for 10 Gigabit Ethernet with a Latency of 433 ns* (2022, DOI: 10.1109/VLSI-DAT54769.2022.9768065). [Nasdaq OUCH](https://www.nasdaqtrader.com/Trader.aspx?id=OUCH) establishes the order-entry role. [Taxi](https://github.com/fpganinja/taxi) supplies candidate AXI-stream/FIFO infrastructure; [Corundum](https://github.com/corundum/corundum) supplies a future NIC/application-integration reference. Neither full codebase is a prerequisite for the minimum core. Any adopted component will retain its license/attribution and have its revision recorded in the later implementation.
