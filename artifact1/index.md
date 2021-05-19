## CellBricks Documentation

This site lays out the source code, step-by-step experimental procedure, 
and data for the CellBricks project artifact1, for the 
ACM SIGCOMM 2021 Conference submission.

#### Contents

- one
- two
- three

#### Usage

You do this and then that. [Link](/subpage)

#### Modified Magma Implementation

We forked Magma, an open source cellular core, to evaluate the latency and 
computational overhead associated with the CellBricks SAP protocol.

#### Measurement Setup

The evaluation of the CellBricks architecture relies on measuring application 
performance from two VPN linked docker containers.

#### Real World Network Experiments

To attain a real perofrmance metrics, we evaluated CellBricks on the T-Mobile 
4G LTE network.

#### Results

These are the results (of course to be updated).


Metric | Result
:-- | :---
<sub><strong>pc</strong></sub>            | <sub>program counter</sub>
<sub><strong>rd</strong></sub>            | <sub>integer register destination</sub>
<sub><strong>rsN</strong></sub>           | <sub>integer register source N</sub>
<sub><strong>imm</strong></sub>           | <sub>immediate operand value</sub>
<sub><strong>offset</strong></sub>        | <sub>immediate program counter relative offset</sub>
<sub><strong>ux(reg)</strong></sub>       | <sub>unsigned XLEN-bit integer (32-bit on RV32, 64-bit on RV64)</sub>
<sub><strong>sx(reg)</strong></sub>       | <sub>signed XLEN-bit integer (32-bit on RV32, 64-bit on RV64)</sub>
<sub><strong>uN(reg)</strong></sub>       | <sub>zero extended N-bit integer register value</sub>
<sub><strong>sN(reg)</strong></sub>       | <sub>sign extended N-bit integer register value</sub>
<sub><strong>uN[reg + imm]</strong></sub> | <sub>unsigned N-bit memory reference</sub>
<sub><strong>sN[reg + imm]</strong></sub> | <sub>signed N-bit memory reference</sub>



### Sponsors

- [Intel](https://www.intel.com/)
- [VMware](https://www.vmware.com/)
- [Ericsson](https://www.ericsson.com/)
- [National Science Foundation](https://www.nsf.gov/)

This project is part of the [Netsys](https://netsys.cs.berkeley.edu) Labratory at UC Berkeley.
