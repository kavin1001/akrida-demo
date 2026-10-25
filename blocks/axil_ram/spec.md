# AXI4-Lite RAM: block specification

## 1. Overview

`axil_ram` is a single-port memory exposed as an AXI4-Lite slave. A master
writes and reads 32-bit words over a standard AXI4-Lite interface. The block
has no registers other than the memory array and no interrupts.

```
          +--------------------------------------+
  AW ---> |  write address  \                    |
  W  ---> |  write data      >-- write ctrl --+  |
  B  <--- |  write response /                 v  |
          |                             [ memory ]|
  AR ---> |  read address   \                 |  |
  R  <--- |  read data       >-- read ctrl <--+  |
          +--------------------------------------+
```

## 2. Parameters

| Name | Default | Meaning |
| --- | --- | --- |
| DATA_WIDTH | 32 | Data bus width in bits |
| ADDR_WIDTH | 16 | Byte address width in bits |
| STRB_WIDTH | DATA_WIDTH/8 | Number of byte lanes |
| PIPELINE_OUTPUT | 0 | 1 adds one register stage on the read data path |

The verification target uses the defaults: 32-bit data, 16-bit addresses,
4 byte lanes, no output pipeline.

## 3. Ports

| Port | Dir | Width | Description |
| --- | --- | --- | --- |
| clk | in | 1 | Clock. All signals are sampled on the rising edge. |
| rst | in | 1 | Synchronous reset, active high |
| s_axil_awaddr | in | ADDR_WIDTH | Write address (byte address) |
| s_axil_awprot | in | 3 | Write protection type (ignored) |
| s_axil_awvalid | in | 1 | Write address valid |
| s_axil_awready | out | 1 | Write address ready |
| s_axil_wdata | in | DATA_WIDTH | Write data |
| s_axil_wstrb | in | STRB_WIDTH | Write byte strobes |
| s_axil_wvalid | in | 1 | Write data valid |
| s_axil_wready | out | 1 | Write data ready |
| s_axil_bresp | out | 2 | Write response |
| s_axil_bvalid | out | 1 | Write response valid |
| s_axil_bready | in | 1 | Write response ready |
| s_axil_araddr | in | ADDR_WIDTH | Read address (byte address) |
| s_axil_arprot | in | 3 | Read protection type (ignored) |
| s_axil_arvalid | in | 1 | Read address valid |
| s_axil_arready | out | 1 | Read address ready |
| s_axil_rdata | out | DATA_WIDTH | Read data |
| s_axil_rresp | out | 2 | Read response |
| s_axil_rvalid | out | 1 | Read data valid |
| s_axil_rready | in | 1 | Read data ready |

## 4. Functional requirements

### Memory and addressing

- **REQ-MEM-1** The memory holds 2^(ADDR_WIDTH - log2(STRB_WIDTH)) words of
  DATA_WIDTH bits (16,384 words of 32 bits with the defaults).
- **REQ-MEM-2** Addresses are byte addresses. The word index is the address
  with the low log2(STRB_WIDTH) bits dropped. The low address bits do not
  affect which word is accessed.
- **REQ-MEM-3** Every word reads as zero before it is first written.
- **REQ-MEM-4** Reset does not clear the memory contents.
- **REQ-MEM-5** Every address in the range is valid. Writes to one word never
  change any other word.

### Writes

- **REQ-WR-1** A write is accepted only once both the write address and the
  write data are valid. The slave may wait for both before raising
  AWREADY and WREADY.
- **REQ-WR-2** For each byte lane i, the byte is written only when WSTRB[i]
  is 1. Lanes with WSTRB[i] = 0 keep their previous value.
- **REQ-WR-3** Each accepted write produces exactly one write response with
  BRESP = OKAY (2'b00).
- **REQ-WR-4** A read issued after a write response has been received
  returns the newly written data.

### Reads

- **REQ-RD-1** Each accepted read address produces exactly one read data
  beat with RRESP = OKAY (2'b00).
- **REQ-RD-2** RDATA holds the full word at the addressed location.

### Handshake rules (AXI4-Lite)

- **REQ-HS-1** Once BVALID or RVALID is raised it stays high, with stable
  BRESP, RDATA and RRESP, until the master raises the matching READY.
- **REQ-HS-2** BVALID is never high in a cycle before the one in which
  both the write address and write data handshakes complete. It may be
  high in that same cycle.
- **REQ-HS-3** RVALID is never high in a cycle before the one in which the
  read address handshake completes. It may be high in that same cycle.
- **REQ-HS-4** The slave must accept new transactions when the master holds
  BREADY or RREADY low for a long time and then releases them: no
  deadlock under back-pressure.
- **REQ-HS-5** Address and data may arrive in either order and in different
  cycles; the slave must handle any skew between AWVALID and WVALID.
- **REQ-HS-6** While rst is high, AWREADY, WREADY, BVALID, ARREADY and RVALID
  are all low.

### Ignored inputs

- **REQ-IGN-1** AWPROT and ARPROT do not affect behavior.

## 5. Out of scope

Exclusive access, burst transactions and error responses (SLVERR, DECERR)
are not supported. AXI4-Lite has no bursts.
