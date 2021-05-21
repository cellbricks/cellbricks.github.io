## CellBricks Documentation

This artifact lays out the source code, step-by-step experimental procedure, 
and data for the CellBricks project, ACM SIGCOMM 2021 Conference submission.

Go back to the CellBricks [homepage](/)

#### Contents

- Overview
- Protocol Summary
- Experiment Phases
    - Prototype Implementation and Evaluation
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

### Prototype Performance Evaluation
 
#### Prototype Testbed

We prototype CellBricks using existing open-source cellular platforms. 
The prototype includes four components: UE(s), the base station (eNodeB), the cellular core, and our broker implementation (brokerd). 
Our testbed has two x86 machines: one acts as UE and the other as bTelco (eNodeB + EPC). We connect each machine to an SDR device ([USRP B205-mini](https://www.ettus.com/all-products/usrp-b205mini-i/)) which provides radio connectivity between the two machines. 

On each machine, we run an extended [srsLTE](https://github.com/cellbricks/srsLTE) suite with the UE machine 
runs the srsUE stack and the eNodeB machine runs the srsENB stack. 
We extend srsLTE with the UE-side changes mentioned in the paper. 
For the cellular core and broker, we run an extended [Magma](https://github.com/cellbricks/magma) implementation. 
The two main components we extend are the access gateway (AGW) and the orchestrator (Orc8r). 
We extend Magma’s AGW to support our secure attachment protocol: we define new NAS messages and handlers and implement 
these as extensions to Magma’s AGW and srsUE. We implement the broker service (called brokerd) as part of Magma’s Orc8r 
component deployed on AWS. Brokerd maintains a database of subscriber profiles (called SubscriberDB) and implements the 
secure attachment protocol, processing authentication requests from bTelcos’.

#### SAP Latency Measurements

We measure the end-to-end latency due to our attachment protocol, 
measured from when the UE issues an attachment request to when attachment completes.
we instrumented the relevant components of our prototype – Access Gateway (AGW), SubscriberDB, Brokerd, eNodeB and UE 
– to measure the processing delay at each.

In our experiments, the UE, eNodeB, and AGW are always located in our local testbed and we run experiments 
with the subscriber database (SubscriberDB) and Brokerd either hosted on Amazon EC2 or our local testbed. 
For each setup, we repeat the same attachment request using both unmodified Magma and Magma with our modifications to implement CellBricks. 
We repeat each test 100 times and report average performance. 

### Application Performance Evaluation on CellBricks

#### Virtual Machine Experiments

To minimize network related performance impact, we setup virtual machines in the Amazon Web Services
(AWS) US-WEST-1 region datacenter and evaluated application performance during cellular handovers 
when TCP (control) or MPTCP (experimental) is in use. The following are utilized:

- 2 virtual machines based on `TODO: ami-111222333`
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

- 2 virtual machines based on `TODO: ami-111222333`
- 2 laptops running Ubuntu 18.04, one [MPTCP enabled](https://github.com/cellbricks/mptcp)
- A wireguard VPN connection established between each VM and laptop (two VM laptop pairs).
- An Open Virtual Switch (OVS) tunnel setup inside the wireguard tunnel
- 4 docker containers based on `silveryfu/celleval`, one on every end of the OVS tunnels
- 2 Cellular dongles, one attached to each laptop
- Modified QCSuper on each laptop `TODO: how were the handover triggers setup?`

The key difference in the physical setup is now the server-client pair is between an AWS VM and a laptop. 
What remain the same are the tunnel and docker setups. We use [QCSuper](https://github.com/cellbricks/emulation/tree/master/ho_proxy/QCSuper), 
an open source Qualcomm baseband logger to detect cellular handovers. Upon these triggers, we emulate a CellBricks SAP handover in a 
similar fashion to the VM-VM approach.

#### Results

The following are application results from the real world experiments.


| Application         |            | iPerf: Avg. Throughput |       | SIP: MOS      |       | Video: Avg. Quality Level |       | Web: Avg. Load Time |       |
|---------------------|------------|------------------------|-------|---------------|-------|---------------------------|-------|---------------------|-------|
| Unit                |            | mbps                   |       | 1-5/excellent |       | level                     |       | sec.                |       |
| Route / Time of Run |            | D                      | N     | D             | N     | D                         | N     | D                   | N     |
| Suburb              | MNO        | 1.25                   | 17.27 | 4.38          | 4.38  | 1.96                      | 4.91  | 4.78                | 1.81  |
|                     | CellBricks | 1.20                   | 16.85 | 4.35          | 4.33  | 1.98                      | 4.91  | 4.96                | 1.76  |
| Downtown            | MNO        | 1.14                   | 16.54 | 4.30          | 4.33  | 2.03                      | 4.94  | 5.12                | 1.89  |
|                     | CellBricks | 1.11                   | 15.41 | 4.25          | 4.32  | 1.97                      | 4.94  | 5.22                | 1.89  |
| Highway             | MNO        | 1.10                   | 11.38 | 4.34          | 4.34  | 1.95                      | 4.89  | 5.05                | 1.86  |
|                     | CellBricks | 1.11                   | 12.42 | 4.27          | 4.30  | 1.97                      | 4.90  | 5.18                | 1.80  |
| Overall             | MNO        | 1.16                   | 15.46 | 4.34          | 4.35  | 1.98                      | 4.91  | 5.00                | 1.86  |
|                     | CellBricks | 1.14                   | 14.99 | 4.29          | 4.31  | 1.97                      | 4.92  | 5.13                | 1.83  |
| % Change            | -          | -1.74                  | -3.09 | -1.16         | -0.92 | -0.51                     | +0.20 | +2.57               | -1.63 |



### Sponsors

- [Intel](https://www.intel.com/)
- [VMware](https://www.vmware.com/)
- [Ericsson](https://www.ericsson.com/)
- [National Science Foundation](https://www.nsf.gov/)

This project is part of the [NetSys](https://netsys.cs.berkeley.edu) Laboratory at UC Berkeley.
