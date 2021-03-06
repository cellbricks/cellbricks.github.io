## SIGCOMM'21 Artifact

This artifact lays out the source code, step-by-step experimental procedure, and data for the CellBricks project, as part of the ACM SIGCOMM 2021 Conference submission.

<!-- Go back to the CellBricks [homepage](/) -->

#### Content

- [Prototype Performance](#prototype-performance)
- [Emulation over the Internet](#emulation-over-the-internet)
- [Traces and Results](#traces-and-results)

### Prototype Performance

#### Testbed Setup

![Testbed Diagram](/testbed.png)

We prototype CellBricks using existing open-source cellular platforms. The prototype includes four components: UE(s), the base station (eNodeB), the cellular core, and our broker implementation (brokerd). Our testbed has two x86 machines: one acts as UE and the other as bTelco (eNodeB + EPC). We connect each machine to an SDR device which provides radio connectivity between the two machines.

Equipment List
- SDR: [USRP B205-mini](https://www.ettus.com/all-products/usrp-b205mini-i/)
- Antennas: [VERT900 Antenna](https://www.ettus.com/all-products/vert900/)

On each machine, we run an extended [srsLTE](https://github.com/cellbricks/srsLTE) suite with the UE machine running the srsUE stack and the eNodeB machine running the srsENB stack. 
We extend srsLTE with the UE-side changes mentioned in the paper. For the cellular core and broker, we run an extended [Magma](https://github.com/cellbricks/magma) implementation. 
The two main components we extend are the access gateway (AGW) and the orchestrator (Orc8r). We extend Magma’s AGW to support our secure attachment protocol: we define new NAS messages and handlers and implement these as extensions to Magma’s AGW and srsUE. We implement the broker service (called brokerd) as part of Magma’s Orc8r component deployed on Amazon Web Services (AWS). Brokerd maintains a database of subscriber profiles (called SubscriberDB) and implements the secure attachment protocol, processing authentication requests from bTelcos’.

We support running the orchestrator in AWS by modifing the [dev_utils.py](https://github.com/cellbricks/magma/blob/master/orc8r/tools/fab/dev_utils.py) 
to allow geteway registration towards a remote (non-local) orchestrator. With this change, setting up Magma's AGW and orchestrator is almost the same as what is described in this [guide](https://magma.github.io/magma/docs/1.1.0/basics/quick_start_guide) with three differences. Firstly, run the orchestrator in an AWS instance, instead of locally. Secondly, in dev_utils.py, change the value of `ORC8R_IP` to the public IP address of the AWS instance, and (optionally) the value of `CERT_DIR` to a directory name where you would like your gateway certificates to reside. Lastly, in the Magma VM, modify the addresses of `controller.magma.test`, `bootstrapper-controller.magma.test` and `fluentd.magma.test` in `/etc/hosts` to the public IP address of the AWS instance.

As for srsLTE, one could run `srsUE` and `srsENB` in the same way as unmodified srsLTE, as described [here](https://github.com/cellbricks/srsLTE/blob/master/README.md). For latency measurements, there are two changes we make on `srsUE`'s usage. 
Firstly, we add two options: `nas.is_bt` for CellBricks attachment benchmarking and `nas.bm_s6a` for standard attachment 
benchmarking. Secondly, to evaluate attachment latency, we add a keyboard input `d` to start a UE-initiated detachment. When either 
`nas.is_bt` or `nas.bm_s6a` is on, `srsUE` will attach right after the detachment finishes, which allows us to repeatedly measure 
UE's attachment latencies. 

More information about the testbed setup can be found in the documents in each code repository.

#### SAP Latency Measurements

We measure the end-to-end latency due to our attachment protocol, measured from when the UE issues an attachment request to when attachment completes.
We instrumented the relevant components of our prototype – Access Gateway (AGW), SubscriberDB, Brokerd, eNodeB and UE – to measure the processing delay at each.

In our experiments, the UE, eNodeB, and AGW are always located in our local testbed and we run experiments with the subscriber database (SubscriberDB) and Brokerd either hosted on Amazon EC2 or our local testbed. For each setup, we repeat the same attachment request (with keyboard input `d`) using both unmodified Magma (`srsUE` with `nas.bm_s6a` on) and Magma with our modifications to implement CellBricks (`srsUE` with `nas.is_bt` on). 
We repeat each test 100 times and report average performance. Results are shown in Figure 4 of the paper.

### Emulation over the Internet

We evaluated CellBricks on the T-Mobile 4G LTE network. This corresponds to the data in Table 1 and Figure 5 of the paper. 

There are two VM-laptop pairs, one for MPTCP and the other for TCP. We also  use [QCSuper](https://github.com/cellbricks/emulation/tree/master/ho_proxy/QCSuper), an open source Qualcomm baseband logger to detect cellular handovers. Upon these triggers, we emulate 
a CellBricks SAP handover in a similar fashion to the VM-VM approach.

- 2 virtual machines based on `ami-06b93cd6edfe1ee9f`, used as the application server.
- 2 laptops running Ubuntu 18.04, one [MPTCP enabled](https://github.com/cellbricks/mptcp), used as the UE.
- 2 LTE to USB Adapters: ZTE MF820B 4G LTE USB
- 2 SIM cards: T-Mobile Prepaid SIM Card (Unlimited Talk, Text, and Data in USA for 30 Days)

Plug in an LTE dongle to each laptop with an active cellular sim (in our case T-Mobile). This will 
provide the internet connection during the tests. Now, follow the above instructions in the "Virtual Machine Experiments" section, except replace the client VM with the physical laptops. On one laptop, disable MPTCP (and make sure it is activated on the other).

```bash
sudo sysctl -w net.mptcp.mptcp_enabled=0
# 0 for off, 1 for on
```

#### Configure the tunnels

**Virtual Setup**
![VM to VM Emulation Diagram](/emulation_dia.png)

There are two hosts, the laptop (client) and cloud VM (server).

On each host, clone the CellBricks emulation repo:
```bash
git clone https://github.com/cellbricks/emulation.git
```

On each host, generate a new public/private keypair for wireguard:
```bash
wg genkey | tee privatekey | wg pubkey > publickey
```

Then, on each host, modify or create `/etc/wireguard/wg0.conf`

Server:
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

Client:
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

On the server, set the environmental variable from the client:
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
start a docker container on each host which will run the application we're testing:

On the server :
```bash
sudo docker run --name uec --privileged -itd --mac-address 00:00:00:00:00:10 silveryfu/celleval
```

Client:
```bash
sudo docker run --name uec --privileged -itd --mac-address 00:00:00:00:00:20 silveryfu/celleval
```

In both containers, update:
```bash
sudo docker exec -it uec /bin/bash
# and then run:
sudo apt update; sudo apt upgrade
```

Also, both containers may have the same IP at the moment, so change the client IP to something else on the subnet 172.17.0.0/16. For example, from inside the docker container:
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

One more thing before running applications, we want to tune the MPTCP variables to match TCP throughput. Modify the tune-mptcp.sh script in the /emulation directory:
* Identify cpus (`lscpu`) and change [TBD 1] in tune-mptcp.sh.
* Identify irq: `dstat -i -N eth0` and change [TBD 2] in tune-mptcp.sh.

Then run the script: `./ tune-mptcp.sh`

#### Run WAN Experiments

Now we are ready to benchmark applications. From inside the containers, you may run applications that communicate with the other container and can change the IP as desired to simulate a cell tower handover. There are a number of scripts in the `/emulation/ho_proxy` folder to automate this process.  See Github README [here](https://github.com/cellbricks/emulation/tree/master/ho_proxy).

When we simulate a handover in the CellBricks protocol, we drop the IP of the client docker container, wait for the associated latency from the SAP, and then establish a new IP on the client container. We measure how the application responds to this handover when the VMs run TCP and compare to when running MPTCP. We measured performance impact using VM to VM experiments as shown in Figure 6 of the paper.

With the VPN and OVS tunnels created, docker containers ready, we may run applications in the containers like before. However, we will trigger handovers based on real network handovers. For example, to benchmark iPerf3:

On both servers:
```bash
sudo docker exec -it uec /bin/bash
iperf3 -S
```

On the client machines, cd into `/emulation/ho_proxy`:
```bash
# In one terminal window
iperf3 -c 172.17.0.2 -i 0.1

# In a second terminal window
python3 proxy.py
```

The above completes the setup for the emulation. More information about the Emulation setup and how to run each application are included in the code repositories.

We pick three representative routes in the downtown, suburban, and highway areas of our geographic region (Berkeley), as shown in the map below. We repeatedly drive along each route with the above setup, each independently running the same application; and we run tests during the day and the midnight-to-dawn period.

![Routes](/routes.png)

#### Study the impact of attachment latency
Finally, to study the impact of attachment latency (Figure 6 of the paper), we set up virtual machines in the AWS US-WEST-1 region datacenter and evaluated application performance during cellular handovers when TCP (control) or MPTCP (experimental) is in use. The setup is similar to the emulation, except that we replace the client machine (the laptop) with a VM in the cloud. Both VMs are running in the same region (US-WEST-1).

On the server, login to the docker container and start the iPerf3 server
```
sudo docker exec -it uec /bin/bash
iperf3 -S
```

Then on the client, cd into the `/emulation/ho_proxy` directory and run:
```
python3 sim.py iperf
```

This will run iPerf3 for 20 second runs, and perform 3 handovers each run, with varying handover latencies as defined in `sim.py`. See [sim.py](https://github.com/cellbricks/emulation/blob/master/ho_proxy/sim.py) for more information.

As the goal is to evaluate MPTCP vs TCP performance with cellular handover, we may turn MPTCP on or off with:
```bash
sudo sysctl -w net.mptcp.mptcp_enabled=0
# 0 for off, 1 for on
```

### Traces and Results

We share the traces collected in our experiments as well as the code to parse the traces into figures in this public Google drive [folder](https://drive.google.com/drive/folders/1t1kQEDJn8gRUAbwv_qhpAwqSdU7BnNWs?usp=sharing). Specifically, the `raw` folder contains emulation traces 
and prototype performance results that we present in the paper. The `notebooks` folder contains scripts for parsing raw traces and results. 
For example, [wan-iperf-by-route.ipynb](https://drive.google.com/file/d/13L_dwDIr9zuOjOps1QeBt_ixFalss8mH/view?usp=sharing) parses iperf traces 
collected in our emulation experiments and generates Figures 5, 6 and 10 of the paper; [prototype-latency.ipynb](https://drive.google.com/file/d/1Eeig7t3VEKxSIxHwX2tE7_wlK4-popDi/view?usp=sharing) parses attachment logs collected in our prototype experiments and generates Figure 4 of the paper. The `figs` folder contains all the figures in the paper.

[NetSys](https://netsys.cs.berkeley.edu) Laboratory at UC Berkeley.
