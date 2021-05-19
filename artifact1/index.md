## CellBricks Documentation

This artifact lays out the source code, step-by-step experimental procedure, 
and data for the CellBricks project, ACM SIGCOMM 2021 Conference submission.

#### Contents

- Overview
- Protocol Summary
- Experiment Phases
    - Magma Modification and Closed Circuit Experiments
    - Application Performance
        - Virtual Machine Experiments
        - Real World Network Experiments


#### Overview

The CellBricks project re-examines the cellular provider landscape. Currently, a small number of 
mobile network operators, or MNOs, dominate the cellular provider industry. Small players and new 
entrants are challenged by the vast infrastructure costs needed to expand service. The CellBricks 
protocol aims to significantly reduce the barrier of entry by decoupling physical infrastructure 
from user management and billing.

There are three entities in CellBricks, the user (UE), broker (B), and bTelco (T). The overall 
architecture is as follows: ... (can expand later to as much as we want)

You do this and then that. [Link](/artifact1/subpage)

### Experiment Phases

#### Magma Modification

We forked Magma, an open source cellular core, to evaluate the latency and 
computational overhead associated with the CellBricks SAP protocol.

#### Closed Circuit Latency Measurements

The evaluation of SAP on dedicated bTelcos to brokers in different geographic regions.

### Application Performance on CellBricks

#### Virtual Machine Experiments

To minimize network related performance impact, we setup virtual machines in the Amazon Web Services
(AWS) US-WEST-1 region datacenter and evaluated application performance during cellular handovers 
when TCP (control) or MPTCP (experimental) is in use. The following are utilized:

- 2 virtual machines based on `ami-111222333 TODO`
- A wireguard VPN connection established between VMs
- An Open Virtual Switch (OVS) tunnel setup between VMs inside the wireguard tunnel
- 2 docker containers based on `silveryfu/celleval` that attach to the OVS tunnel

The goal is for a networked application to run between two machines in an environment where we 
may immediately cut the connection and reestablish after some delay associated with the SAP. In 
the setup, the wireguard VPN connects the two VMs, and the protocol is highly resilient to handovers, 
IP changes, and other interference in the network. This helps control for other factors that may 
otherwise affect application performance, which becomes even more important in the real world 
experiments. With a stable tunnel setup, a second, very lightweight tunnel in OVS creates a virtual  
switch attached on both VMs. Finally, the two docker containers, one on each VM, are spun up and 
attached to the OVS tunnel, meaning the two containers can communicate with each other. We may pick 
one VM to be the server and the other to be the client.

When we simulate a handover in the CellBricks protocol, we drop the IP of the client docker container, 
wait for the associated latency from the SAP, and then establish a new IP on the client container. We 
measure how the application responds to this handover when the VMs run TCP and compare to when running 
MPTCP.

See Github README [here](https://github.com/cellbricks/emulation/tree/master/ho_proxy) for step by step instructions.

#### Real World Cellular Network Experiments

To attain real performance metrics, we evaluated CellBricks on the T-Mobile 4G LTE network. The 
setup is similar to the purely virtual machine experiments, but utilizes the following:

- 2 virtual machines based on `ami-111222333 TODO`
- 2 laptops running Ubuntu 18.04, one [MPTCP enabled](https://multipath-tcp.org/pmwiki.php/Users/HowToInstallMPTCP?)
- A wireguard VPN connection established between each VM and laptop (two VM laptop pairs).
- An Open Virtual Switch (OVS) tunnel setup inside the wireguard tunnel
- 4 docker containers based on `silveryfu/celleval`, one on every end of the OVS tunnels
- 2 Cellular dongles, one attached to each laptop
- Modified QCSuper on each laptop `TODO how was the handover triggers setup on the laptop again?`

The key difference in the physical setup is now the server-client pair is between an AWS VM and a laptop. 
What remain the same are the tunnel and docker setups. We use QCSuper, an open source Qualcomm baseband 
logger to detect cellular handovers. Upon these triggers, we emulate a CellBricks SAP handover in a 
similar fashion to the VM-VM approach.

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

This project is part of the [NetSys](https://netsys.cs.berkeley.edu) Laboratory at UC Berkeley.
