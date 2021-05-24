## CellBricks Documentation

This artifact lays out the source code, step-by-step experimental procedure, 
and data for the CellBricks project, ACM SIGCOMM 2021 Conference submission.

Go back to the CellBricks [homepage](/)

#### Contents

- Prototype Implementation and Evaluation
- Application Performance
    - Virtual Machine Experiments
    - Real World Network Experiments

### Prototype Performance Evaluation
 
#### Prototype Testbed

![Testbed Diagram](/testbed.png)

We prototype CellBricks using existing open-source cellular platforms. 
The prototype includes four components: UE(s), the base station (eNodeB), the cellular core, and our broker implementation (brokerd). 
Our testbed has two x86 machines: one acts as UE and the other as bTelco (eNodeB + EPC). We connect each machine to an SDR device  which provides radio connectivity between the two machines.

Equipment List
- SDR: ([USRP B205-mini](https://www.ettus.com/all-products/usrp-b205mini-i/))
- Antennas: VERT900 Antenna

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
We repeat each test 100 times and report average performance. Results are shown in Figure 4 of the paper.

### Application Performance Evaluation on CellBricks

#### Virtual Machine Experiments

To minimize network related performance impact, we setup virtual machines in the Amazon Web Services
(AWS) US-WEST-1 region datacenter and evaluated application performance during cellular handovers 
when TCP (control) or MPTCP (experimental) is in use. The following prerequisites are needed:

- 2 AWS EC2 instances based on `ami-06b93cd6edfe1ee9f` (only available in US-WEST-1)

One VM will serve as the server, and the other as the client (pick one to serve each role). 
Next, set up a VPN connect between the virtual machines.

On each virtual machine, clone the CellBricks emulation repo:
```bash
git clone https://github.com/cellbricks/emulation.git
```

One each virtual machine, generate a new public/private keypair for wireguard:
```bash
wg genkey | tee privatekey | wg pubkey > publickey
```
On each VM, modify or create `/etc/wireguard/wg0.conf`

VM1 (Server):
```
[Interface]
PrivateKey = [server's private key from previous step]
ListenPort = 55107
Address = 192.168.4.1

[Peer]
PublicKey = [client's public key from previous step]
AllowedIPs = 192.168.4.2/32, 192.168.4.3/32
PersistentKeepalive = 25
```

VM2 (Client):
```
[Interface]
Address = 192.168.4.2
PrivateKey = [client's private key from previous step]
ListenPort = 51820

[Peer]
PublicKey = [server's public key from previous step]
AllowedIPs = 192.168.4.1/32
Endpoint = [HOST2]:55107
PersistentKeepalive = 25
```

On the server, set the environmental variable from the client VM:
```bash
WG_REMOTE=192.168.4.2
```

Now, cd into the ho_proxy directory of the emulation repo we cloned earlier:
```bash
cd emulation/ho_proxy
```

And bring up the tunnel on both machines:
```bash
make tun
```

If you want to bring down the tunnel after running experiments, run:
```bash
make del-tun
```

At this point, both the WireGuard tunnel and OVS tunnels have been created. Now we will 
start a docker container on each VM which will run the application we're testing:

On the server VM:
```bash
sudo docker run --name uec --privileged -itd --mac-address 00:00:00:00:00:10 silveryfu/celleval
```

Client VM:
```bash
sudo docker run --name uec --privileged -itd --mac-address 00:00:00:00:00:20 silveryfu/celleval
```

In both containers, update:
```bash
sudo docker exec -it uec /bin/bash
# and then run:
sudo apt update; sudo apt upgrade
```

Also, both containers may have the same IP at the moment, so change the client IP to something 
else on the subnet 172.17.0.0/16. For example, from inside the docker container:
```
ifconfig eth0 172.17.0.5 netmask 255.255.0.0 broadcast 0.0.0.0
route add default gw 172.17.0.1
```

On the server, you can verify the container's IP. For the applications scripts, they assume a 
server container UP of 172.17.0.2:
```
ifconfig eth0 172.17.0.2 netmask 255.255.0.0 broadcast 0.0.0.0
route add default gw 172.17.0.1
```

One more thing before running applications, we want to tune the MPTCP variables to match TCP throughput. 
Modify the tune-mptcp.sh script in the /emulation directory:
* Identify cpus (`lscpu`) and change [TBD 1] in tune-mptcp.sh.
* Identify irq: `dstat -i -N eth0` and change [TBD 2] in tune-mptcp.sh.

Then run the script: `./ tune-mptcp.sh`

Now we are ready to benchmark applications. From inside the containers, you may run applications that 
communicate with the other container and can change the IP as desired to simulation a cell tower handover. There 
are a number scripts in the `/emulation/ho_proxy` folder to automate this process. For example to run iPerf3, we could:

On the server VM, login to the docker container and start the iPerf3 server
```
sudo docker exec -it uec /bin/bash
iperf3 -S
```

Then on the client VM, cd into the `/emulation/ho_proxy` directory and run:
```
python3 sim.py iperf
```

This will run iPerf3 for 20 second runs, and perform 3 handovers each run, with varying handover latencies as defined in 
`sim.py`. See [sim.py](https://github.com/cellbricks/emulation/blob/master/ho_proxy/sim.py) for more information.

As the goal is to evaluate MPTCP vs TCP performance with cellular handover, we may turn MPTCP on or off with:
```bash
sudo sysctl -w net.mptcp.mptcp_enabled=0
# 0 for off, 1 for on
```

When we simulate a handover in the CellBricks protocol, we drop the IP of the client docker container, 
wait for the associated latency from the SAP, and then establish a new IP on the client container. We 
measure how the application responds to this handover when the VMs run TCP and compare to when running 
MPTCP. We measured performance impact using VM to VM experiments as shown in Figure 6 of the paper.

See Github README [here](https://github.com/cellbricks/emulation/tree/master/ho_proxy).

#### Real World Cellular Network Experiments

To attain real performance metrics, we evaluated CellBricks on the T-Mobile 4G LTE network. This corresponds 
to the data in Table 1 and Figure 5 of the paper. The setup is similar to the purely virtual machine experiments, but the 
key difference in the physical setup is now the server-client pair is between an AWS VM and a laptop. There 
are two VM-laptop pairs, one for MPTCP and the other for TCP. We also  use [QCSuper](https://github.com/cellbricks/emulation/tree/master/ho_proxy/QCSuper), 
an open source Qualcomm baseband logger to detect cellular handovers. Upon these triggers, we emulate 
a CellBricks SAP handover in a similar fashion to the VM-VM approach.

- 2 virtual machines based on `ami-06b93cd6edfe1ee9f`
- 2 laptops running Ubuntu 18.04, one [MPTCP enabled](https://github.com/cellbricks/mptcp)
- 2 LTE to USB Adapters: ZTE MF820B 4G LTE USB
- 2 SIM cards: T-Mobile Prepaid SIM Card (Unlimited Talk, Text, and Data in USA for 30 Days)

Plug in an LTE dongle to each laptop with an active cellular sim (in our case T-Mobile). This will 
provide the internet connection during the tests. Now, follow the above instructions in the "Virtual 
Machine Experiments" section, except replace the client VM with the physical laptops. On one laptop, 
disable MPTCP (nad make sure it is activated on the other).
```bash
sudo sysctl -w net.mptcp.mptcp_enabled=0
# 0 for off, 1 for on
```

With the VPN and OVS tunnels created, docker containers ready, we may run applications in the containers 
like before. However, we will trigger handovers based on real network handovers:

On both servers:
```bash
sudo docker exec -it uec /bin/bash
iperf3 -S
```

On the client machines, cd into `/emulation/ho_proxy`:
```bash
# In one terminal window
TODO: run application
# In a second terminal window
python3 proxy.py
```

Now you may walk / drive around town with the two laptops and emulate the CellBricks protocol!

[NetSys](https://netsys.cs.berkeley.edu) Laboratory at UC Berkeley.
